# PhIPSeq Plate Designer v4 — dithered plate layouts

`plate_manifest_tool_v4.html` extends the v3 designer so a list of subjects can be laid
out as **side-by-side duplicates** and **dithered across a user-defined number of plates**,
instead of being packed into contiguous cohort blocks.

Open the file in a browser and upload a manifest built from
`PhIPSeq_Manifest_Template_v4.xlsx` (v3 manifests still load unchanged).

## Why dither

In block fill, a cohort occupies whole plates. Any plate-level batch effect —
reagent lot, incubation, thermocycler position — is then perfectly confounded with
cohort, and cannot be separated from the biology at analysis time. Dithering spreads
each group evenly over a window of plates, so plate becomes a nuisance variable that
can be modelled rather than a confounder.

## Autofill options

| Control | Meaning |
|---|---|
| **Layout mode** | `Dither groups across plates` (new) or `Block fill` (v3 behaviour, unchanged). |
| **Group / stratify by** | The variable balanced across plates: cohort, diagnosis, `dither_group`, sample type, sex, subject ID, or none. Only fields carrying data in the uploaded manifest are offered. |
| **Dither across N plates** | Window size. Each block of N consecutive plates receives a proportional share of every group. N = plate count spreads each group over the whole run. |
| **Well placement** | How samples are arranged *within* a plate. `Ordered` (default) gives each group a contiguous run of column pairs — the plate reads as solid blocks, which is what makes hand pipetting quick. `Scatter` randomises which well pair a sample lands in, breaking up row/column position effects too; use it once a robot is doing the work, since a machine does not care about layout. `Sequential` fills A1→H10 in deal order, leaving groups interleaved. |
| **Sample order within group** | `Sequential` (default) gives each plate a consecutive run of every group in manifest order — plate 1 takes Cohort A 1–17, plate 2 takes 18–34, and so on — so tubes are picked up in rack order. `Randomised` gives each plate a seeded random subset instead; per-plate counts are identical either way. |
| **Random seed** | Every random choice comes from this seed, so the same inputs always produce the same layout. Written to the Design Log sheet. |
| **Replicates per sample** | 2 = side-by-side duplicate (40 samples/plate), 1 = singlet (80 samples/plate). |
| **Keep subjects whole** | All samples from one `subject_id` (e.g. longitudinal draws) land on the same plate, so within-subject comparisons are never split across a batch. |

## Algorithm

1. **Units.** A unit is one sample, or — with *keep subjects whole* on — every sample
   belonging to one subject. Each sample in a unit consumes one replicate chunk.
2. **Chunks.** Wells are cut into replicate chunks from the canonical column-pair order
   (`A1+A2, B1+B2, … H9+H10`) and a chunk is only offered if *every* well in it is free.
   A duplicate is therefore always physically side-by-side, even when some wells were
   placed by hand first. Columns 11–12 are never touched.
3. **Plate count.** The plan uses only as many plates as the sample count requires, so
   dithering never leaves half-empty plates behind. Spare plates are kept as overflow.
4. **Quota.** Plates are cut into windows of N. Every group is apportioned across the
   windows in proportion to window capacity (largest remainder), so no group can drift
   into one end of the run. Plates beyond what the sample count needs get no quota and
   serve only as overflow, keeping short runs compact.
5. **Deal.** Within a window the groups are dealt one after another in manifest order —
   all of the first group, then all of the second — each group split across the window's
   plates as **consecutive runs** sized in proportion to the room each plate has left.
   Because the units arrive in manifest order, a plate receives a contiguous block of
   each group; selecting `Randomised` shuffles the group first, which turns the same
   contiguous slicing into a random subset per plate.
6. **Placement.** Units are written into that plate's chunk list. This step decides
   arrangement only — which plate a sample belongs to is already fixed by step 5, so
   changing placement never disturbs the dither balance. In `ordered` mode the plate's
   units are sorted by group and then by manifest order, so each group forms one solid
   run down the column pairs and tubes are picked in rack order within it. In `scatter`
   mode the chunk list is shuffled first.

The preview shows the realised **group × plate cross-tab** with a per-group spread
(min–max per plate) before anything is applied; nothing is written until *Apply*.

## Prefilling

`buildFreeChunks()` reads the live `plateAssignments`, so hand-placed wells simply do
not appear as free chunks, and `eligible` excludes any sample already on a plate.
Autofill therefore composes with manual placement in any order, and `applyAutofill()`
merges rather than replaces.

Because chunks are cut from the canonical pair order and kept only when *every* well
in them is free, occupying one half of a pair makes the other half unreachable —
filling it would break the side-by-side duplicate invariant. `strandedWells()` finds
those wells and the preview reports them per plate rather than letting the capacity
disappear silently.

## Mixed sample types

Different sample types need different prep, so the tool treats a mixed plate as a
condition worth flagging rather than a detail to look up.

`plateComposition()` counts wells per `sample_type` on a plate. A plate is **mixed**
when more than one *non-control* type is present — assay controls and mAbs are
excluded from that test, since they always differ from the samples and occupy the
reserved columns 11–12, and counting them would make every plate read as mixed.

Three consequences:

1. In ordered well placement, sample type sorts ahead of group, so each type forms a
   contiguous block of wells.
2. The Lab Plate Maps `PLATE n` label gains the composition, prefixed `MIXED PREP`
   where it applies. The label still begins with `PLATE n` and the plate number is
   still the first integer on the row, so plate detection on re-import is unaffected.
3. On a mixed plate only, every Lab Plate Maps cell is suffixed with its type in
   square brackets — `SER-001 [serum]`. Uniform plates are left untagged.

Analysis Plate Maps is excluded from 2 and 3 on purpose: it is the machine-read tab,
so its cells stay bare `tiu_id` values and its headers stay plain `PLATE n`.

## Repeated samples

`repeatSpec()` reads the optional `repeat_across_plates` column: `all`/`yes`/`true`
for every plate, or a list like `1,3,5`. Such samples are held out of the ordinary
dealing pool and placed once per target plate, before anything else, each pinned to a
fixed slot index so it lands in the same well on every plate — where the plate's own
geometry allows, since a plate with a different control area has a different first
free well. Plate-count arithmetic subtracts the bridging load from each plate's
capacity before deciding how many plates the run needs.

They remain available in the sample list after placement (like controls), and report
under their own `↻ bridging (repeated)` row so a cohort's per-plate count stays true.

Bridging can also be switched on per sample in the app, with the **↻** button on a
sample card — `manualBridge` holds those `tiu_id`s. An app-bridged sample is never
consumed from the pool, and is excluded from autofill entirely (both from the dealt
pool and from automatic bridging placement) on the grounds that you are placing it
by hand; the preview lists which samples that applies to. The **Show placed samples**
checkbox reveals samples already on a plate, which is how an already-placed sample
gets bridged after the fact.

On export, **any** identifier appearing on more than one plate takes a `-Pn` suffix,
on both the Lab and Analysis tabs — two plates means two wells and two measurements.
Single-plate samples are untouched.

## Control area

`reservedOn(plate)` returns the plate's explicit well set if one was defined,
otherwise the run-wide default of columns 11–12 when the toolbar checkbox is on.
Presets cover columns 11–12 and none; any other shape comes from the current well
selection. The
canonical slot list spans all six column pairs, so what a run may use is decided
entirely by the reservation rather than being hardcoded — freeing columns 11–12 on a
plate raises its capacity from 40 to 48 duplicated samples. `buildFreeChunks()`
excludes both assigned and reserved wells, so autofill can never write into a control
area, and manual assignment there is still limited to `assay control` samples.

## Round-tripping a filled manifest

`restoreAssignments()` rebuilds `plateAssignments` from an uploaded workbook,
preferring the `Sample Instances` sheet (`source_tiu_id` + `plate` + `wells`, which is
unambiguous) and falling back to parsing the Analysis grid, stripping any `-Pn`
suffix. Lab Plate Maps is never used as a source because its cells may carry
`[sample_type]` tags. `restoreChangeLog()` reads any existing `Change Log` rows and
marks them `prior: true` so later exports distinguish them from the current session.
`inferBridgingFromLayout()` adds anything sitting on more than one plate to
`manualBridge`, so a restored layout keeps those samples in the pool.

Edits go through `applyWellEdit()`, which refuses to run without a template user,
resolves the replicate pair via `wellsOfSampleOnPlate()`, and appends to `changeLog`
before mutating `plateAssignments`. `exportFileName()` strips a trailing
`_YYYYMMDD-HHMM` and `_filled` before appending a fresh stamp, so repeated exports do
not accumulate suffixes.

## Replicate layout

`getSampleSlots(layout)` decides the order wells are consumed, and a replicate takes
consecutive slots — so one setting controls both how a duplicate sits and which way
the fill travels:

| layout | first pairs | shape |
|---|---|---|
| `pair-down` (default) | A1+A2, B1+B2, C1+C2 | side by side, down a column pair |
| `pair-across` | A1+A2, A3+A4, A5+A6 | side by side, along a row |
| `stack-down` | A1+B1, C1+D1, E1+F1 | stacked, four samples per column |

`buildFreeChunks()` and `strandedWells()` both take the layout, so pair alignment and
stranded-well detection follow whichever shape is in use.

## Export styling

The stock SheetJS community build reads styles but does not write them, so the tool
loads `xlsx-js-style` — a drop-in fork of 0.18.5 with the same `XLSX` global and utils
API, plus cell-style output.

`buildPlateSheet()` collects a `styleTargets` list while filling the value grid, then
applies styles once the sheet object exists. Well fills use `tintARGB(colour, 0.78)`
against the cohort colour as the font, mirroring the soft-fill/saturated-text
relationship on screen. `autoFitColumns()` measures only the well-grid rows: the plate
label and the colour key sit in column A and would otherwise stretch the row-letter
column across the sheet, and Excel spills their text over the empty cells to the right
anyway. Column A is pinned narrow, and the colour key puts its swatch in A with the
cohort name in B.

Only the Lab tab is coloured. Analysis Plate Maps gets column widths and nothing else.

## Export

Existing sheets are unchanged. A **Design Log** sheet is added when an autofill plan was
applied, recording mode, group variable, window size, seed, replicate count, subject
handling, plates used, the group × plate composition, and the sample types and control area of each plate — enough to reproduce the
layout exactly and to describe it in a methods section.

## Template

`PhIPSeq_Manifest_Template_v4.xlsx` adds one optional column, `dither_group`, for an
explicit stratification variable when cohort/diagnosis are not the right axis (study
site, collection batch, freezer). It is optional and additive — v3 manifests load
unchanged, and `dither_group` simply appears as another *group by* choice when populated.
