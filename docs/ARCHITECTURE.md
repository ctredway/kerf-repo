# Kerf architecture

Kerf is deliberately one self-contained HTML file. That's not laziness — it's the deployment model: no build step, no dependencies beyond three.js from a CDN, runs from a file:// path or any static host, and the whole app can be reviewed in one read. If it ever outgrows this, the module boundaries below are where it splits.

## Module map (top to bottom of the `<script>`)

| Section | Responsibility |
|---|---|
| **Parser** (`parseGcode`) | GRBL-dialect tokenizer + modal-state interpreter. Produces tessellated micro-segments (all internal units mm), lint issues, tool-change list, tool metadata harvested from CAM comments (Vectric, Fusion 360, generic fallback), toolpath sections. Pure function; fully testable headless. |
| **Dynamic checks** (`dynamicChecks`) | Settings-dependent lints (work-area, stock depth, lateral rapids below Z0) recomputed when settings change without re-parsing. |
| **Viewer** | three.js scene: work-area grid (cells flush to the table edge), toolpath as two `LineSegments` (solid cuts / dashed rapids) with per-vertex progress coloring updated by delta, tool marker, view cube rendered in a scissored second pass over the same canvas. Custom orbit controls. |
| **Playback** | Time-indexed simulation: cumulative-time binary search, per-source-line stepping, virtualized code panel synced to the current line. |
| **Serial layer** (`SERIAL`, `handleRx`, `parseStatus`) | Web Serial connection, line assembly, status-report parsing (state, MPos/WPos/WCO, pins, feed/speed), `$$`/`$#`/`$I` auto-query, GRBL error/alarm translation. `handleRx` routes acks by priority: probe → jog → job. |
| **Machine profile** (`PROFILE`) | Persisted settings (localStorage + JSON export/import, including Carbide Motion `shapeoko.json` import). Single source for BitSetter position, spindle behavior, jog prefs, traverse height, run options. |
| **Controller config** | `$` settings store, machine-model detection from travel values, auto-sync of work area / rapid rate / jog rates. |
| **Jogging** (`JOGC`) | Step jogs and ack-clocked continuous jog (≤3 segments in flight, each `ok` releases the next), commanded-position travel clamp on homed machines, pin-trip guard (edge-triggered), keyboard capture scoped to the jog modal. |
| **Job streaming** (`JOB`, `buildJobList`) | Pre-built send list from the source file: cleaned lines, M6 markers with next-XY lookups, injected spin-up dwells, injected pre-stop retracts, optional start-high pre-position. Character-counting protocol (≤100 of GRBL's 128 RX bytes in flight). Error → auto feed-hold. |
| **BitSetter probing** (`PROBE`) | Sequential two-touch probe state machine (fast seek → back off → slow probe), reference at job start, per-tool-change delta applied via volatile `G43.1`. Validates the stored position against controller travel before any motion. |
| **UI shell** | Header (e-stop, state/spindle chips, hold/resume, icon actions), sidebar tabs (Code / Checks / Machine), jog modal, tabbed settings modal, footer transport + DRO. |

## Design principles

1. **The G-code file is never modified.** Everything Kerf adds — dwells, retracts, pre-positions, probe sequences — is injected into the *send list* at runtime. The file on disk stays portable and safe in any other sender.
2. **Runtime intelligence belongs to the sender.** The file can't know where the tool is, whether the machine is homed, or what the real travel is; Kerf can, so behaviors that depend on runtime state (start/stop-high, travel preflight, tool-length offsets) live here, never in CAM post hacks.
3. **Flow control everywhere.** Both the job stream (character counting) and continuous jog (ack clocking) are paced by the controller's acknowledgements. Nothing blind-fires on a timer into the RX buffer.
4. **Trust is explicit.** Machine-coordinate moves (`G53`) require `homedSeen` — a Home→Idle transition observed this session. Alarms and resets revoke it. Untrusted position degrades features gracefully rather than guessing.
5. **Volatile over persistent for automated offsets.** Tool-length compensation uses `G43.1` (cleared by reset) rather than rewriting `G54`, so no interrupted job can permanently corrupt the user's work zero.
6. **Safety states are loud and low-tech.** The e-stop is text, not an icon. The lock banner explains itself. Preflight dialogs state exactly what will be injected. First probe of a session prints its full command sequence.

## Testing

There is no formal test suite in-repo yet, but every protocol-level behavior (parser geometry, streaming flow control, jog clamps, probe sequencing and delta signs, M6/M5/dwell injection ordering) was developed against headless Node simulations of a GRBL controller. Porting those harnesses into `tests/` is welcome work.
