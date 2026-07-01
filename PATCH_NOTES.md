# ACIDRATCHET-303 v5.5 — Cutoff / Resonance Mapping Patch

## Purpose

This patch keeps the existing ACIDRATCHET-303 sound engine intact and only adjusts the filter control mapping.

The reported behaviour was:

- Cutoff only starts gripping around ~45%.
- Sweet spot sits too high, around ~75%.
- The upper range does not open enough.
- Resonance must not become a sterile continuous whistle.

## Changed DSP Lines

### Previous cutoff model

```js
var baseHz = 34 * Math.pow(150, sm.cutoff); // 34..5100 Hz
```

This mapping leaves too much of the lower knob range musically inactive.

### New cutoff model

```js
var cut = sm.cutoff;
var cutCurve = 0.075 + 0.925 * Math.pow(cut, 0.64);
var baseHz = 30 * Math.pow(245, cutCurve);
```

## Expected Behaviour

- Cutoff starts reacting earlier.
- 35–65% becomes more useful.
- Former 75% sweet spot should move closer toward 55–65%.
- Top range opens more clearly.
- Resonance remains capped and compressed to avoid pure self-oscillation.

## Resonance Protection

The resonance cap was raised slightly for bite, but still limited:

```js
if (k > 4.22) k = 4.22;
if (fc > 7200) k *= Math.sqrt(7200 / fc);
```

This keeps the high range more alive while preventing permanent piercing whistle tones.

## Test Protocol

1. Load INIT.
2. Set Saw wave.
3. Set Resonance 70–82%.
4. Set Env Mod 60–80%.
5. Set Decay 40–60%.
6. Sweep Cutoff slowly from 20% to 90%.

Expected:
- First grip should appear before 40%.
- Strong acid zone should appear around 55–70%.
- 80–90% should open the top without turning into static sine whistle.

## Rollback

If the patch over-opens the filter, reduce:

```js
Math.pow(cut, 0.64)
```

to:

```js
Math.pow(cut, 0.70)
```

If it still reacts too late, reduce it to:

```js
Math.pow(cut, 0.58)
```

## Files

- `index.html` — patched playable file
- `PATCH_NOTES.md` — this document
