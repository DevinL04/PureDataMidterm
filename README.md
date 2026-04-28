# CMPM 125 Mock Midterm - NES Sound Chip Emulation

**Due: 4/28/2026 @ 11:59pm PST**

Build a Pure Data patch that emulates the Ricoh 2A03 (NES) sound chip and syncs with a Unity scene over OSC.

---

## Assignment requirements

- Use **all** NES sound channels **except** the sample player:
  - Pulse 1 (square wave)
  - Pulse 2 (square wave)
  - Triangle
  - Noise
- Sequencing
- Per-voice volume control
- Clean audio (no clicks/pops)
- PD patch and Unity scene synced via OSC
- Instructions + comments in the patch

---

## Files in this repo

| File | Purpose |
|---|---|
| `MockMidterm.pd` | **Main patch** - OSC receiver + instantiates all sounds + mixer -> dac~ + instructions |
| `pickup.pd` | Cube-pickup sound. Uses NES Pulse 1 + Pulse 2. Triggered by `/unity/trigger`. |
| `wall.pd` | Wall-collision noise burst. Uses NES Noise channel. Triggered by `/unity/colwall`. |
| `sequencer.pd` | Step sequencer playback. Uses NES Triangle channel. Tempo controlled by `/unity/tempo`, start/stop by `/unity/playseq`. |
| `MockMidterm_reference.pd` | **Professor's example - do not edit.** Reference only. |
| `BallDemo_MockMidterm/` | Unity project |

### How the 4 NES channels are covered

Instead of one file per channel, each sound file uses whichever channels fit that sound. All 4 required channels still end up represented:

| NES Channel | Lives in |
|---|---|
| Pulse 1 | `pickup.pd` |
| Pulse 2 | `pickup.pd` |
| Triangle | `sequencer.pd` |
| Noise | `wall.pd` |
| (Sample player) | omitted per assignment |

---

## Who owns what

| File | Owner |
|---|---|
| `pickup.pd` | Devin |
| `wall.pd` | Devin |
| `sequencer.pd` | Joey |
| `MockMidterm.pd` | Coordinated - only one person edits at a time |

Only touch your own files. If you need to change the main patch, message the other person first.

---

## Conventions we agreed on

### Clean audio - every voice uses `[vline~]` for amplitude
Every sound output goes through a `[vline~]` driving a `*~` for amplitude shaping. No direct amplitude changes on `*~` - that causes clicks. Envelope messages follow the reference pattern:
```
0.85 100 , 0 500 500     (attack to 0.85 over 100ms, then release to 0 over 500ms with 500ms delay)
```

### Internal routing uses `send`/`receive` pairs
Each voice abstraction listens for its own OSC events directly via `[r busname]`. This keeps the main patch clean - we don't have to draw wires from the OSC receiver to every voice.

Example: the pulse voice can include `[r oscpulse1]` internally and respond to trigger events without any wiring in the main patch.

### Comments in every file
Each voice file starts with a `[text]` or `[comment]` explaining what it does and what its inlets expect. Required by the assignment, also saves time for whoever reads it next.

---

## OSC setup (copied from professor's reference)

- **Listen port:** 8000 (UDP binary, `netreceive -u -b`)
- **Send port:** 8001 to `127.0.0.1`
- **Address scheme:** `/unity/<event>`
- **Events already defined by reference:** `trigger`, `tempo`, `playseq`, `colwall`
- **Events we'll add as needed:** one per voice if we want Unity to trigger each channel individually.

Use vanilla PD objects: `netreceive -u -b` + `oscparse`. **No mrpeach dependency.** Works out of the box on PD 0.51+.

The OSC receive subpatch in the reference is fair game to copy into our `MockMidterm.pd` - it's boilerplate plumbing, not graded sound-design work. Just add our own comments to show we understand it, and add any new `route` branches for events we add.

---

## What the reference already shows us

The professor's reference uses only **three voices** (sine, noise, triangle) - it is NOT complete. Our job is to extend it to meet the assignment's four-channel requirement and to use proper NES sound generation.

- **Cube pickup sound:** two `osc~` sines. We need to replace this with real pulse waves (this is where Pulse 1 and Pulse 2 come in).
- **Wall collision:** `noise~` through an envelope. Good starting point for the noise channel.
- **Sequencer playback:** triangle wave built from two `phasor~` (one inverted, summed, offset). This is a compact way to make a triangle - use it as a basis for `triangle.pd`.

### NES pulse wave tips
A pulse wave is a square wave with a variable duty cycle (12.5% / 25% / 50% / 75%). In PD, build one with `[phasor~]` compared against a threshold value - changing the threshold changes the duty cycle. That's the NES-authentic sound.

---

## OSC message reference (existing)

| Address | Data | Effect |
|---|---|---|
| `/unity/trigger` | int 0-8 | Trigger cube pickup sound at semitone offset (C major scale) |
| `/unity/colwall` | bang | Trigger wall collision noise |
| `/unity/tempo` | int (ms) | Set sequencer metro interval |
| `/unity/playseq` | 0 or 1 | Stop / start sequencer |

Add more as needed and update this table.

---

## Git workflow

**Main rule: only one person edits the same file at a time.** Especially `MockMidterm.pd`.

Before editing:
```
git pull
```

After editing:
```
git add <your files>
git commit -m "describe what you added"
git push
```

Tell the other person in chat when you're done so they can pull.

### If something breaks
- `git log --oneline` to see history
- `git show <commit>:<file>` to see an old version
- Every commit is a checkpoint - commit small and often

---

## Suggested order of work

1. **One of us** copies the OSC receive subpatch from `MockMidterm_reference.pd` into `MockMidterm.pd`, adds comments, sets up the mixer (inlets -> `*~` for per-sound volume -> sum -> `dac~`), and instantiates empty `[pickup] [wall] [sequencer]` boxes. Commit + push.
2. **After pull**, we each open our own files and start building sounds. Devin on `pickup.pd` + `wall.pd`, partner on `sequencer.pd`.
3. Commit after every working feature - one sound that plays cleanly on trigger = one commit.
4. Test each sound in isolation before wiring into the main patch.
5. Test from Unity: send OSC messages from the scene and confirm each sound responds correctly.
6. Final pass: clean up, add instructions text to the main patch, double-check for clicks/pops.

---

## Known gotchas

- **Pure Data line endings:** Windows PD may save with CRLF, Mac with LF. Git will warn but it's harmless. If diffs look like the whole file changed, that's why.
- **Abstractions must live in the same folder as the main patch** (or in PD's search path). Don't move them into subfolders.
- **Pure Data caches abstractions** - if you edit `pulse.pd` while `MockMidterm.pd` is open, you may need to close and reopen the main patch to pick up changes.
- **OSC from Unity** - make sure Unity is sending to port 8000 (UDP), matching what PD listens on.
