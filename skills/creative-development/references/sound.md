# Sound

Sound makes an interactive piece feel physical: wind that follows the cursor, a hit on click, a swell on each transition. Treat it like motion. Design it, drive it from the same frame loop, and make it work on phones.

## Synthesize It

Build the sounds with Web Audio instead of downloading files. A noise buffer through a band-pass is wind, detuned saws through a low-pass with a slow LFO are a drone, short sine blips on a pentatonic scale are grains, and a sine sweeping from 140 Hz to 38 Hz with filtered noise is an impact. Nothing to load, nothing to decode, and every parameter can follow the pointer.

- Route every voice into one `master` gain, through a `DynamicsCompressor`, with a convolver reverb on a send. Fade `master` with `setTargetAtTime` to mute and unmute instead of stopping nodes.
- Drive continuous voices from the frame loop with `setTargetAtTime` (wind gain and filter from cursor speed, filter Q from intensity). Spend discrete voices (grains) from a budget that grows with speed, so fast movement plays more notes.
- Use the real pointer speed, not an idle "ghost" path, so the piece is quiet when nobody is touching it.

## Schedule on the Audio Clock

Anything rhythmic (a sequencer, a beat, notes on a grid) is scheduled ahead on `audioContext.currentTime`, never fired from timers at the moment it should play. From the frame loop, or a 25 ms interval, schedule every step that falls within the next 0.12 to 0.15 seconds. Swing odd steps by delaying them a fraction of a step. After the tab was hidden, restart from the next bar instead of catching up on every missed step.

## One Value Drives Picture and Sound

Pick the motion value the visitor controls and feed it to both sides. Scroll velocity sets the playback rate and a waveshaper's drive while it bends the planes; the spin speed of an object opens and closes one low-pass filter over the whole mix. The sound then feels caused by the motion, not played over it.

## Listening to Audio

When the visuals react to sound:

- Tap the `AnalyserNode` before the `master` gain, so muting doesn't starve the visuals.
- Normalize with a running peak instead of fixed thresholds, and detect onsets with spectral flux against an adaptive mean plus a multiple of the standard deviation.
- Open the microphone with `echoCancellation`, `noiseSuppression` and `autoGainControl` all off, or the browser flattens what you want to see.
- For offline renders, precompute the whole track's spectrum and waveform into data textures (one row per group of frames), so any shader can read any moment.

## Start It from a Gesture

Browsers only allow audio after a user gesture, so create the `AudioContext` on the first one and call `resume()` inside that handler. Start muted with an "unmute" hint and a toggle button. Any click or tap on the page unmutes, and only the button mutes again.

The gesture has to be one the browser counts as user activation: `click`, `keydown`, `touchend`, `mousedown`, and `pointerup` for touch and pen. `pointerdown` and `touchstart` don't count.

## Phones

Test sound on a real iPhone. Three things break there that never break on desktop:

1. **iOS doesn't dispatch `click` for taps on non-interactive elements**, such as a full-screen canvas, so a `window` click listener never fires. Listen to `touchend` and to `pointerup` with `pointerType !== 'mouse'` as well. Unmute only switches sound on, so receiving several events for one tap is harmless.
2. **The silent switch mutes Web Audio**, because iOS routes it through the ringer like a notification. Set `navigator.audioSession.type = 'playback'` (Safari 17+) before creating the context, and it plays through the switch like video does.
3. **The context gets interrupted** by calls, Siri or another app taking audio, and only a gesture can restart it. On `touchend`, resume the context if sound is enabled and its state isn't `running`.

The toggle button must stop `pointerdown`, `pointerup` and `touchend` from bubbling. Otherwise the page-level unmute switches sound on and the button's own click switches it straight back off.

```ts
for (const type of ['pointerdown', 'pointerup', 'touchend']) {
  this.button.addEventListener(type, (event) => event.stopPropagation())
}

window.addEventListener('click', this.onUnmute)
window.addEventListener('pointerup', this.onPointerUp) // calls onUnmute when pointerType isn't 'mouse'
window.addEventListener('touchend', this.onUnmute)
```

## Housekeeping

- Suspend the context on `visibilitychange` when the tab is hidden, and resume it on return only if sound was on.
- Keep the button's `aria-pressed` and `aria-label` ("Turn sound on" / "Turn sound off") in sync with the state.
- Hide the unmute hint with CSS once the button is pressed (`.sound[aria-pressed='true'] ~ .hint`). On touch screens (`@media (hover: none)`) the hint says "Tap", everywhere else "Click".
