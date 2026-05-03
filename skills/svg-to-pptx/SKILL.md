---
name: svg-to-pptx
description: Rebuild a self-contained SVG figure (typically the AutoFigure-Edit pipeline output) as an editable PowerPoint deck and extract its inlined icons as standalone PNGs. Use when a generated SVG looks roughly right but needs hand polishing of arrows, text, or layout, or when icon assets are needed for slides, posters, or follow-up papers.
---

# SVG to PPTX Rebuild

## When to Use

Trigger on any of these:

- A user has a generated SVG figure that mostly works but needs hand polishing (arrows, text positions, missing labels).
- A user wants to edit a generated figure in PowerPoint or Keynote.
- A user needs the embedded icon assets as separate files for reuse in slides, posters, or revised papers.
- The SVG is self-contained, meaning icons are inlined as `<image href="data:image/png;base64,...">`.

Skip when the SVG references external image files. The script only handles inlined base64.

## What This Script Does

Reads an SVG and walks its element tree once. For each tag:

| SVG Tag | PowerPoint Output |
|---------|-------------------|
| `<image>` with `data:image/...;base64,...` | Decoded to PNG/JPG/WEBP under `assets/icons/<id>.<ext>`, then placed via `add_picture` at scaled coordinates |
| `<rect>` | `RECTANGLE` or `ROUNDED_RECTANGLE` (when `rx>0`), with fill, stroke, stroke width preserved |
| `<circle>` | `OVAL` |
| `<line>` | `MSO_CONNECTOR.STRAIGHT` |
| `<polygon>` | Freeform line segments via `FreeformBuilder.add_line_segments` |
| `<text>` | Textbox with `text-anchor` mapping to paragraph alignment, italic detection over nested `<tspan>`, and `transform="rotate(...)"` mapped to shape rotation |

Slide width is fixed at 13.333 inches (standard 16:9 widescreen). Slide height is computed from the SVG `viewBox` aspect ratio so the figure fills the slide without distortion. EMU per SVG unit and Pt per SVG unit are derived from these.

## What This Script Does Not Do

| Skipped | What to Do in PowerPoint |
|---------|--------------------------|
| `<path>` (bezier curves, dashed alignment lines, curved arrows) | Redraw with `Insert -> Shapes -> Connector` |
| Arrowhead markers (`marker-end="url(#...)"`) | PowerPoint connectors carry built-in arrow endcaps; toggle in the line format pane |
| Linear and radial gradients | Apply via `Format Shape -> Fill -> Gradient` |
| Drop shadows, blur filters | Apply via `Format Shape -> Effects` |
| Dashed strokes | Set in the line format pane |
| Super and subscript inside `<tspan>` (used for `T+`, `T-`, `r+`, `r-`) | Select the offending characters and toggle Format `->` Superscript |
| `transform="translate/scale/skew"` on `<g>` groups | Either flatten the SVG before running, or extend the script to push a transform stack |

## Run

From the project root:

```
python skills/svg-to-pptx/scripts/extract_to_pptx.py [svg_path] [output_dir]
```

Defaults: `todo/final.svg` and `todo/`.

Outputs:

- `<output_dir>/assets/icons/<svg_id>.<ext>` (one file per inlined image, named by the SVG `id` attribute when present)
- `<output_dir>/final_rebuild.pptx` (single slide)

The script is idempotent. Re-running overwrites both targets.

Stdout reports counters of the form `{'image': 22, 'rect': 74, 'text': 101, 'polygon': 10, 'circle': 14, 'line': 5, 'path_skipped': 26, 'other_skipped': 10}`. A non-zero `path_skipped` is expected for any figure with curved arrows or dashed alignment lines.

## Dependencies

```
pip install python-pptx lxml
```

`Pillow` is not strictly required by the script itself but is useful for inspecting the extracted PNGs.

## Tuning Knobs

Inside the script:

- `SLIDE_W_INCHES = 13.333`. Change when targeting a non-default slide template (for example 10 for 4:3, 13.333 for widescreen).
- Font size scaling: `fs_pt = fs_units * pt_per_unit`. The conversion is `pt_per_unit = (SLIDE_W_INCHES * 72) / svg_w`. If text appears systematically too small or too large in the resulting deck, multiply `fs_pt` by a constant or change the slide width.
- Minimum stroke width: `Emu(6350)` (about 0.5 pt). Hairline strokes from the SVG would otherwise vanish at PowerPoint's default zoom.
- Text box width estimation: `len(text) * fs_units * 0.55`. Sans-serif characters are typically 0.5 to 0.6 em wide; adjust if a font with very different metrics is in use.

## Known Caveats

- Tiny SAM3 detections (1 to 5 pixel boxes) become tiny PNGs that are valid but useless. Delete them from the slide manually.
- Stacked text in the source SVG can produce overlapping textboxes in PowerPoint. Move them apart by hand.
- The Bianxie or older provider versions sometimes produce SVG with `xlink:href` instead of `href`. The script handles both.

## Extending

Each branch in the main loop is independent. To add support for a new element type, add an `elif tag == "your_tag":` block and use the python-pptx API directly. For path support, `FreeformBuilder.add_line_segments` is the easiest extension point for straight polylines. Bezier control points need direct XML manipulation through the `_element` accessor.

## Files

- `scripts/extract_to_pptx.py` (the converter)
