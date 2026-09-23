# 454 Control vs Carbide Motion — behavioural differences

Notes for documentation. Anyone arriving from Carbide Motion knows its habits, so anywhere
454 Control behaves differently needs saying out loud rather than discovering mid-job.

Status: living document. Add to it whenever a difference shows up in testing.

---

## Probing

**BitZero probe is two-stage.** Carbide Motion lowers the tool slowly until the circuit
closes. 454 Control searches down at 150 mm/min, backs off 1.5 mm, then re-probes at
40 mm/min and takes *that* reading. Same accuracy (the measurement is the slow pass), about
five times quicker from a 10 mm start. The BitSetter uses the same fast-then-slow pattern.

*Why it matters:* the first descent looks alarmingly quick to someone expecting the slow
Carbide Motion creep.

**Probe failures leave the machine locked.** If the tool never reaches the plate, GRBL raises
ALARM:5 and locks up. 454 Control explains what happened and offers Unlock ($X).

**Start height still matters:** 2–10 mm above the plate.

## Tool changes

**BitSetter reference is measured at job start**, not when Z is zeroed. Carbide Motion
measures the tool when you set Z zero, specifically so the tool can't be swapped in between.

*Consequence:* zero Z with the bit that will be in the spindle when you press Start.
This is the one real gap; worth closing.

**Tool-change position defaults to the front centre** of the machine, worked out from the
controller's travel limits. Carbide Motion parks about 100 mm left of the BitSetter.
Configurable in 454: front centre, front left, front right, back centre, or a captured spot.

**The prompt is a modal**, not a button in a panel, and it names the tool.

## Spindle

**Spin-up dwell is configurable and defaults to 10 s.** Carbide Motion inserts about 2.5 s
(G4 P2 plus G4 P0.5), which suits a trim router. A VFD spindle takes longer to reach speed.
Neither is in the G-code: both senders add it.

**454 Control reports where the bit waits** during spin-up, and warns if the file starts the
spindle after descending.

## Starting and ending a job

**Homing is required.** A job won't start without it, because machine coordinates are needed
for lifting clear, parking and the BitSetter.

**"Start & stop high"**: travel at the top of Z before plunging, at the start and after each
tool change, and lift clear before every spindle stop.

**Every job ends the same way:** lift clear, stop the spindle, then move off the part — to the
park position if one is captured, otherwise back to the job's XY zero. This happens whatever
the file does or forgets to do.

**Lifts never move down** and are skipped when the tool is already clear.

## Stopping

**End job sends a controller reset**, which stops the spindle where it stands with no lift.
Hold is the gentler pause. (Under review: a controlled stop that lifts and parks first.)

**Leaving the page mid-job is caught** by a browser prompt, and the app tries to reset the
controller on the way out.

## Recovery

**Job recovery reconstructs state** (units, motion mode, feed, spindle, tool, position) and
approaches carefully, and you can Hold or Stop at any point during the approach. Carbide
Motion's restart-at-line reportedly gives no way to bail out once it starts moving.

**The resume line is suggested from where the machine actually stopped**, not from the last
line sent, since the controller runs behind the stream.

## The app itself

**It runs in a browser** (Chrome or Edge, Web Serial). Settings live in that browser, not in a
system-wide install, and profiles export to a file.

**No firmware updates or machine setup wizard.** Keep Carbide Motion for those; 454 Control
only reads the controller's $ settings and checks the ones it depends on.

**Quick Actions** are saved G-code snippets, similar in spirit to Carbide Motion's.

**Themes**: light and dark, with a choice of accent colour, shared with 454 Design.
