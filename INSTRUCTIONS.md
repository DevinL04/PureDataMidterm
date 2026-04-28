# NES Sound Chip Emulation, Instructions

**CMPM 125 Mock Midterm**
Devin Lopez and Joey Ferlatt

## What this project does

This is a Pure Data patch that emulates the four melodic channels of the Ricoh 2A03 sound chip (the chip inside the original NES) and connects to a Unity scene over Open Sound Control (OSC). When you play the Unity scene, picking up cubes triggers little NES style pickup chimes, hitting walls plays a noise burst, and a triangle channel sequencer runs in the background and speeds up the more cubes you collect. We left out the sample player channel since the assignment did not require it.

The four channels we built:

* Pulse 1 and Pulse 2, the two square wave channels. We use both of these for the cube pickup sound.
* Triangle, the bass channel. This drives the looping arpeggio sequencer.
* Noise, used for the wall collision sound.

## What you need to run it

* Pure Data (we used vanilla PD, no mrpeach or other externals. Everything is built with stock objects plus the `expr` library that ships with vanilla PD).
* Unity 2020.2.1f1 (the version the BallDemo project was built with. Newer versions probably work but we did not test them).

## How to start everything up

1. Open PD first so it is ready to receive OSC. File, then Open, then `MockMidterm.pd`.
2. In PD, switch to run mode if it is in edit mode (Ctrl+E to toggle. The cursor goes from an arrow to a pointer).
3. Click the DSP toggle (top left, the small square next to "DSP on/off"). It should appear pressed.
4. Raise the volume sliders. They all default to 0 (silent) on patch load. You need:
   * Pickup voice volume slider (left voice strip)
   * Wall voice volume slider (middle voice strip)
   * Sequencer voice volume slider (right voice strip)
   * Master volume slider (far right)
5. Open Unity, load the `BallDemo_MockMidterm` project, open `Assets/_Scenes/minigame.unity`, and press Play.

You should hear the sequencer start immediately, since Unity sends `/unity/playseq 1` when the scene begins. Move the ball with WASD or the arrow keys to trigger the other sounds.

## File layout

| File | What it is |
|------|------------|
| `MockMidterm.pd` | The main patch. This is the only file you open. |
| `pickup.pd` | The cube pickup sound (Pulse 1 + Pulse 2). |
| `wall.pd` | The wall collision sound (Noise). |
| `sequencer.pd` | The 8 step arpeggio (Triangle). |
| `MockMidterm_reference.pd` | Professor's reference patch. Do not edit, do not submit. |
| `BallDemo_MockMidterm/` | Unity project. |

`pickup.pd`, `wall.pd`, and `sequencer.pd` are PD abstractions. When MockMidterm.pd has a box like `[pickup]`, PD looks for `pickup.pd` in the same folder and embeds it. So all four `.pd` files have to stay in the same folder for the patch to work.

## How OSC connects PD and Unity

Unity sends OSC messages over UDP on `127.0.0.1:8000`. The OSC setup is in `OSCHandler.cs` and `MovePlayer.cs` inside the Unity project. PD listens on that same port using `[netreceive]` inside the `pd oscReceive` subpatch. From there, PD parses the messages with `[oscparse]` and `[route]`, then sends them to internal buses via `[s ...]` objects, which the voice abstractions listen to with `[r ...]` objects.

The four messages Unity sends:

| Address | Value | What happens |
|---------|-------|--------------|
| `/unity/trigger` | int 0 to 8 | Plays a pickup at that semitone in C major |
| `/unity/colwall` | int | Plays the wall noise burst |
| `/unity/playseq` | 0 or 1 | Stops or starts the sequencer |
| `/unity/tempo` | int (ms) | Sets the sequencer's metro interval |

## Controls inside the main patch

**Voice volumes.** One slider per voice plus a master, in the row of voice strips. We used a `[hsl]` into `[pack f 20]` into `[line~]` into `[*~]` chain so volume changes ramp over 20 ms instead of jumping. Otherwise you get clicks when you move a slider.

**VOICE PARAMETERS panel** (under the voices). Five sliders that change the sound while it is running. We wired these up using the slider's SEND name field so they write directly to the voice's control buses with no intermediate objects.

* **Pulse 1 duty cycle and Pulse 2 duty cycle** (0 to 1). Changes the timbre of the pickup sound. NES pulse waves can have 12.5%, 25%, 50%, or 75% duty cycles, and you can hear the difference. Smaller duty sounds thinner and buzzier, while 50% is a full square. We used `[phasor~]` into `[expr~ if($v1 < $v2, 1, neg 1)]` (where the second comparison value is a signal) so the slider can change duty live.
* **Pickup pitch sweep amount and time.** Adds a pitch chirp to each pickup trigger. Set the amount to a positive number for a downward chirp, a negative number for an upward chirp. 0 means no chirp at all.
* **Wall low pass cutoff** (Hz). Changes how bright or dark the wall thud sounds. Lower values make it darker and heavier, higher values make it brighter.

**TEST PANEL** (right side). Lets you trigger every voice without running Unity, which was useful while we were building the patch. The numbered messages 0 through 8 trigger pickup at that semitone, the bang button triggers wall, the 1 and 0 messages start and stop the sequencer, and the floatatom sets the sequencer tempo.

## NES techniques we used

A few things we figured out while building this:

* **Pulse waves with variable duty cycle.** We did not know how to do this at first, since the professor's reference uses two `[osc~]` sines, which are not really NES pulses. The trick is `[phasor~]` for the ramp, then comparing it against a threshold value to get a square. We used `[expr~]` to do the comparison because vanilla PD's `<~` did not take a creation argument in our PD version, and `expr~` lets us make the duty controllable.
* **Pitched noise via samphold.** The reference's wall sound is just plain `[noise~]` through an envelope. We replaced this with `[noise~]` going into `[samphold~]` driven by a `[phasor~]`, which is closer to how the actual NES noise channel works. It samples random values at a chosen rate, and the rate determines the perceived pitch. We also sweep the rate from 1500 Hz down to 200 Hz over 80 ms, which gives a "boom" feel instead of a flat hiss.
* **Triangle wave from `[abs~]`.** We tried the reference's two phasor pattern but could not get it to behave, so we used a different approach. `[phasor~]` into `[‐~ 0.5]` into `[abs~]` into `[*~ 4]` into `[‐~ 1]` generates a clean bipolar triangle, since the absolute value of a centered ramp gives you the triangle shape.
* **`[vline~]` envelopes everywhere.** Every voice runs its audio through a `[*~]` whose right input is driven by a `[vline~]` envelope. This avoids clicks and pops when notes start and stop. Same idea for the volume sliders, where we use `[line~]` instead of feeding the slider straight into `[*~]`.
* **Internal buses for OSC routing.** Voices listen via `[r osctrig]`, `[r oscwall]`, and so on, which means the main patch does not need wires from the OSC receiver to every voice. This keeps the main patch a lot cleaner and easier to follow.

## Notes and things to know

* The voice volume sliders default to 0 on every load. We did this on purpose so the patch does not blast at full volume the moment you turn DSP on.
* The first OSC message Unity sends is `/unity/trigger "ready"` (a string). PD's `[route trigger]` floatatom may print a "non numeric" warning when this lands. It is harmless. The actual gameplay triggers send ints.
* If PD says "audio input device number (0) out of range," you can ignore it. That message is about audio input, and our patch only uses output.
* If you change a `.pd` file while MockMidterm.pd is open, you have to close MockMidterm.pd and reopen it for the change to take effect. PD caches abstractions until the parent patch is reloaded.
