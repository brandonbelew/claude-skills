name: bambu-studio
description: Create editable Bambu Studio print-prep projects and Bambu-ready deliverables. Use when working with Bambu Lab printers, Bambu Studio, editable Bambu text/text_info 3MF files, editable CAD source, STL/STEP/3MF exports, slicer settings JSON, filament/process/machine profiles, 3D-printing measurement intake, fit tolerances, print orientation, connected-component/floating-object checks, bed-orientation checks, paired-fit invariants, build-plate fit checks, or repeatable workflows that turn text-editable project files into print-ready Bambu Studio .3mf output.
---

# Bambu Studio

Use this skill to keep Bambu print-prep work editable and repeatable: source CAD and settings JSON are the primary files; STL/STEP/3MF are generated deliverables.

## Core Workflow

1. Preserve or create editable source first:
   - Prefer `source/*.scad` for OpenSCAD-parametric parts when the user is already using OpenSCAD.
   - Prefer `source/*.py` plus CadQuery only when STEP-native parametric modeling is clearly useful.
   - Do not treat exported `.stl` or `.3mf` meshes as the source of truth unless no source exists.

2. Export geometry:
   - Export `.step` when Bambu Studio should preserve higher-quality CAD import.
   - Export `.stl` for broad compatibility and mesh validation.
   - Export geometry per physical part unless the user asks for a combined plate.

3. Validate before slicer packaging:
   - Confirm each part exports successfully.
   - Check manifold/simple status when the CAD tool reports it.
   - Check bounding boxes against the selected printer build volume.
   - Check connected components; more than one component can trigger Bambu floating-object warnings.
   - Check bed orientation: `min_z` should be near `0`, footprint should fit the plate, and bottom-contact faces should exist on the intended print face.
   - For OpenSCAD on Windows/PowerShell, verify `-D` quoting with a tiny probe if part selection behaves oddly.

4. Add Bambu Studio settings as editable text:
   - Store full Bambu Studio CLI settings under `bambu/settings/`.
   - Use `machine.json`, `process.json`, and `filament-*.json` naming.
   - Treat Bambu Studio repo profile JSON as reference material; the CLI expects full exported configs for `--load-settings` and `--load-filaments`, not partial inherited system profiles.

5. Generate Bambu-ready outputs:
   - Use Bambu Studio CLI when available to load geometry/settings and export a `.3mf`.
   - Keep the generated `.3mf` in `outputs/` beside the raw geometry exports and a short assumptions/validation note.

## Recommended Project Layout

```text
project/
  source/
    model.scad
  bambu/
    settings/
      machine.json
      process.json
      filament-pla-basic.json
  outputs/
    part-a.step
    part-a.stl
    part-a.3mf
    NOTES.md
```

For multi-part projects, use stable part names in every layer: `base.scad` selector -> `base.stl` -> `base.3mf` -> notes entry.

For mating parts, record paired-fit invariants in `outputs/NOTES.md`. For sliding panels/cassettes, never change panel size without checking base groove size, lid capture rib span, and slot clearance.

## Bambu Studio CLI Pattern

Use the CLI as a compiler from editable inputs to print-ready output:

```powershell
bambu-studio --orient --arrange 1 `
  --load-settings "bambu/settings/machine.json;bambu/settings/process.json" `
  --load-filaments "bambu/settings/filament-pla-basic.json" `
  --slice 0 `
  --export-3mf outputs/project.3mf `
  outputs/model.step
```

For existing Bambu projects, use `--export-settings` to obtain full editable settings JSON before attempting to synthesize settings from partial profiles.

## 3MF Inspection

Use `scripts/inspect_3mf.py` to list a `.3mf` archive's internal files, identify likely settings/model/thumbnail/metadata entries, and report Bambu editable text candidates found in `Metadata/model_settings.config`. Inspection is useful; hand-authoring Bambu `.3mf` internals is brittle. Prefer generating `.3mf` through Bambu Studio CLI or Bambu Studio itself.

## Build-Volume Checks

Use `scripts/check_stl_bounds.py` before packaging. It reports bounds, build-volume fit, connected components, expected component count, lowest-Z placement, footprint, and bottom-contact triangles/area. Common Bambu volumes:

- A1 mini: `180 x 180 x 180`
- X1/P1/A1 family: `256 x 256 x 256`
- H2D/H2D Pro: `350 x 320 x 325`

If the selected printer is unknown, ask for it or default to the conservative `256 x 256 x 256` only when that matches the user's prior context.

Pass `--expected-components N` for intentional multi-part plates, such as a base plus panel exported together. Default to `1` for single printable parts.

Treat these results as warnings to investigate before slicing:

- `components != expected_components`: possible floating/disconnected object warning in Bambu Studio unless the multi-part plate is intentional.
- `min_z` far from `0`: part may import above/below the bed.
- `bottom_contact_triangles = 0`: likely wrong orientation or only edge/point bed contact.
- footprint larger than plate: part must be rotated, split, scaled, or moved to a larger printer.

## OpenSCAD Notes

When repairing existing OpenSCAD:

- Preserve the existing architecture and make the smallest viable geometry fix.
- Export and validate every requested part rather than stopping at code inspection.
- Use `scripts/export_parts.ps1` as a starting point for batch STL/3MF export of named OpenSCAD selectors.
- On PowerShell, if `-D part=...` selects the wrong part or renders the full layout, probe the quoting before debugging geometry.
- If `openscad` is not on PATH and installers are blocked, use a portable workspace-local OpenSCAD binary when available.

## Editable Text and Labels

OpenSCAD `text()` exports as mesh geometry. It will not remain editable as Bambu Studio text. If the user wants editable labels, either export a blank/label-ready part and add text inside Bambu Studio, or generate a Bambu editable-text `.3mf` with `scripts/generate_editable_text_3mf.py`. Use mesh text only when permanent engraved/raised geometry is desired.

Distinguish MakerWorld/OpenSCAD generator customizability from Bambu Studio editable text. A `.3mf` may describe a customizable generator in `3D/3dmodel.model` metadata while still containing only a mesh. Inspect `Metadata/model_settings.config` first:

- Bambu Studio editable text is indicated by a part-level `<text_info ... />` element in `Metadata/model_settings.config`. Useful attributes include `text`, `font_name`, `font_size`, `thickness`, `embeded_depth`, `rotate_angle`, `text_gap`, `bold`, `italic`, `surface_type`, `hit_mesh`, `hit_position`, and `hit_normal`.
- Editable text may still be under `subtype="normal_part"` and the object file may still contain baked mesh vertices/triangles. Do not classify it as mesh-only when `<text_info ... />` exists.
- If there is no `<text_info ... />`, the part name ends in `.stl`, and `3D/Objects/*.model` contains only vertices/triangles, treat text as mesh-only.
- If generator terms such as `OpenSCAD`, `Add_Bar`, `Base_Thickness`, fonts, or customizable text appear only in the model description metadata, they are not editable Bambu Studio text parameters.
- A known-good Bambu Studio text specimen uses `<text_info text="Codex Test" font_name="Courier New" font_version="2.5" ... font_size="12.000007629394531" thickness="2" ... />` under `Metadata/model_settings.config`.
- Confirmed editable-text recipe: start from a Bambu text `.3mf` project structure, include placeholder or matching mesh geometry in `3D/Objects/object_1.model`, add part-level `<text_info ... />` in `Metadata/model_settings.config`, then open in Bambu Studio. Bambu Studio can double-click/edit this text and regenerate the mesh on edit/save.
- To create an editable text-only `.3mf`, run:

```powershell
python scripts/generate_editable_text_3mf.py `
  --text "LABEL TEXT" `
  --font "Source Han Sans JP" `
  --font-size 12 `
  --thickness 2 `
  --output outputs/label-text.3mf
```

- The bundled generator uses `assets/editable_text_template.3mf` by default and creates a simple 5x7 block-glyph placeholder mesh. For final print geometry, open the result in Bambu Studio, double-click the text, confirm/regenerate, save, then inspect the saved file with `scripts/inspect_3mf.py`.
- Patching only the `text` attribute without regenerating the mesh can leave stale displayed geometry until Bambu Studio regenerates it.

## Generic 3D Printing Intake

Ask for only the measurements needed to unblock the model. Prefer millimeters, calipers, and a labeled sketch or straight-on photos with a ruler in frame.

For enclosures, brackets, adapters, panels, and mounts, gather:

- Overall width x depth x height.
- Mounting hole diameter, hole type, and center-to-center spacing.
- Wall, panel, tab, slot, or mating-part thickness.
- Connector/cable clearance and keep-out zones.
- Intended bed face and visible faces.
- Fit goal: snug, sliding, loose/removable, press-fit, screw-clearance, screw-pilot, or heat-set insert.
- Use case: decorative, electronics enclosure, load-bearing, heat exposure, outdoor/UV, flex/snap tabs, or repeated assembly.

Use conservative default clearances only when the user has not specified fit:

- Sliding fit: about `0.3-0.6 mm` per side.
- Loose removable panel: about `0.5-0.8 mm` per side.
- Press fit, snap fit, threads, and heat-set inserts: ask for material/printer/nozzle or use a test coupon.
- Screw holes: distinguish clearance hole, pilot hole, countersink/counterbore, and heat-set insert diameter.

Before finalizing a part, state the intended bed face, expected support need, visible-face support risk, and any weak layer-direction risk. For existing CAD, preserve the current architecture unless asked to redesign, preserve mating dimensions, and list changed fit-critical dimensions.

Deliver editable source, generated STL/3MF outputs, validation notes, assumptions, printer/build-volume target, and fit-critical dimensions.

## Safety Boundaries

Do not provide Bambu cloud/auth bypass instructions, reverse-engineered network-control workflows, or guidance for evading Bambu Connect or authorization systems. Keep this skill focused on editable local project files, official Bambu Studio CLI workflows, geometry validation, and print-prep packaging.
