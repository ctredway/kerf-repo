# Kerf Design — v0.22.2

Major update to the sketcher: VCarve `.crv` and DXF import, a complete
Edit Vectors toolset, construction geometry, and a proper tool state
machine. Also a long run of geometry-robustness fixes driven by real
production files.

## Suggested commit message (short)

    Kerf Design 0.22.2: DXF import, full edit-vectors suite, tool state machine

    - Import: .crv drawing/toolpath separation; DXF (lines, arcs, circles,
      LWPOLYLINE bulges, splines, ellipses) with scale-to-size and unit detection
    - Edit Vectors: fillet, trim, extend, offset, copy, mirror, rotate, array,
      join, explode; trim now cuts paths/polys, not just lines/arcs/circles
    - Construction geometry (X to toggle), marquee select, layered Esc
    - Tool state machine: one manager owns transitions + VCarve-style check/X
      hover feedback; ends the tool-state-leak bug class
    - Configurable vector / toolpath / construction colors
    - Many geometry fixes: join arc preservation, >180deg bulge rendering,
      tangent-tolerance for rounded DXF coords, concave/convex fillet selection

---

## What changed, in detail

### Import
- **.crv (VCarve) import** now separates *drawing* vectors from *toolpath
  preview* vectors and lets you choose which to bring in (drawing only by
  default), so imported parts no longer arrive tangled with cutter-offset
  ghost geometry.
- **DXF import** added: LINE, CIRCLE, ARC, LWPOLYLINE (including bulge arcs,
  mapped straight onto the native path model), POLYLINE/VERTEX, SPLINE
  (NURBS sampled), and ELLIPSE. Unsupported entity types are counted and
  reported rather than silently dropped.
- **Scale-to-size** on import: type a target width and the drawing scales
  uniformly to hit it; optional move-to-origin; units auto-detected from
  `$INSUNITS` (inch vs mm), confirmable in the import dialog.

### Edit Vectors (new tool group)
- **Fillet** — line/line, line/arc, and arc/arc corners; convex and concave
  (reentrant) corners; live radius entry (no Enter needed); rejects
  zero/negative radii.
- **Trim** — cuts lines, arcs, circles, **and now paths/polys** at crossings;
  click the piece to remove; works in either direction (cut the line at the
  shape, or the shape at the line). Entities with no crossings are never
  silently deleted.
- **Extend** — runs a line or arc out to the next crossing.
- **Offset** — inward/outward parallel copies of lines, arcs, circles, rects,
  polys and paths; winding-independent; chainable.
- **Copy / Mirror / Rotate / Array** — selection transforms; mirror can use a
  construction line as its axis; array does linear (`N@dx,dy`) and circular
  (`N<step`) patterns.
- **Join / Explode** — stitch fragments into connected paths (arc spans
  preserved as bulges) and break paths/polys/rects back into lines and arcs.

### Interaction
- **Construction geometry** — press **X** to toggle any selection to dashed
  reference geometry (Fusion-style). Snaps, dimensions, and acts as a mirror
  axis / trim boundary. Survives save/load and transforms.
- **Marquee select** — left-to-right = window (fully enclosed), right-to-left
  = crossing (touched); shift adds.
- **Layered Esc** — cancels an in-progress operation, then exits to Select,
  then clears the selection.
- **Configurable colors** — separate, persisted colors for vectors, toolpath
  previews, and construction geometry.

### Tool state machine (architecture)
- A single **tool manager** owns the active tool and every transition
  (enter/exit), centralizing prompt, focus, in-progress state and selection
  policy. This removes the class of bugs where state leaked between tools
  (stuck focus, wrong prompt, first-click-eaten, "won't switch tools").
- **VCarve-style hover feedback**: edit tools interrogate the geometry under
  the cursor and show a green check when a click will act or a red X when it
  won't, highlighting the affected geometry. Hover and click share the same
  test, so the cursor never promises an action the click won't perform.
- Adding a new tool is now a registry entry plus its geometry function.

### Geometry robustness (fixes found against real production files)
- **Join** preserves arc geometry correctly (bulge bookkeeping at fragment
  joints, reversals, and loop closure); large trimmed circle-arcs no longer
  collapse onto a connector.
- **tessPath** renders arcs on the correct side for any sweep, including
  >180deg and negative-bulge spans (fixes fillet "hooks" and rendering cusps).
  This hardens every arc-drawing path, not just fillets.
- **Tangent tolerance** widened for rounded DXF coordinates, so trim/extend
  detect crossings that sit within rounding of a segment endpoint (fixes
  circles vanishing on trim, and tangent-junction fillets).
- **Fillet center selection** chooses the geometrically correct solution for
  both convex and concave corners (nearest-tangent-to-corner), no longer
  producing wrong-side fillets.
- **angNorm** hardened against non-finite input (fixes a render freeze on a
  degenerate persisted span).
- Import no longer clobbered by stale localStorage (startup restore separated
  from post-import UI sync).

### Housekeeping
- Visible version string in the header and tab title.
- `docs/LICENSING.md`: open-core charter — the sketcher and all import stay
  MIT; the future CAM engine (drawings -> G-code) will be the commercial
  layer, developed in a separate private repo.
