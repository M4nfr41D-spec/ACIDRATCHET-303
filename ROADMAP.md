
# ACIDRATCHET-303 — Filter Bite Recovery Roadmap

## Why this document exists

The engine has been patched twice (v5.4 → v5.5, see `PATCH_NOTES.md`) and both
patches made the same mistake: they moved the *cutoff mapping curve* around,
but every time resonance started to bite, another clamp was added to flatten
it back down. The result is a synth that measures "303-shaped" on paper but
never actually snarls.

This roadmap does the opposite. It removes every piece of code whose sole job
is to neutralize or symmetrize resonance, removes clamps that exist only to
keep the sound "safe" rather than because the math needs them, and replaces
the filter core with a topology that can self-oscillate and growl the way a
real diode-ladder 303 (and its best modern clone, the **Behringer TD‑3‑MO /
TE‑3‑MO**) does. Nothing here reduces headroom by adding *more* limiting —
every step either removes a limiter or replaces an external clamp with an
internal, physically-motivated nonlinearity.

**Target reference:** Behringer TD‑3‑MO / TE‑3‑MO diode-ladder filter —
resonance that can be driven into full self-oscillation (clean sine tracking
the cutoff pitch), bite/growl that increases continuously with resonance
instead of hitting a ceiling, and a cutoff sweep that stays "alive" at the
top of the knob range instead of being damped back down.

All line numbers below refer to `index.html` at commit `2640364372e87d76a10071d8c15e2cda27a67d96`
(function `makeVoice`, the AudioWorklet DSP core — the single source of truth
shared by the worklet and the ScriptProcessor fallback).

---

## Root-cause inventory (what's currently strangling the sound)

| # | Location | Code | Effect |
|---|----------|------|--------|
| 1 | `index.html:508` | `if(k>4.22)k=4.22;` | Hard resonance ceiling. No matter how the knob/env/accent push `k`, it's clipped to a fixed number → true self-oscillation is architecturally impossible. |
| 2 | `index.html:509` | `if(fc>7200)k*=Math.sqrt(7200/fc);` | Frequency-dependent resonance damping. The higher the cutoff, the more resonance is silently drained — this is the "symmetrizing" behavior: it forces the top of the cutoff range to sound tamer than the middle, which is backwards from a real diode ladder (and from the TD‑3‑MO). |
| 3 | `index.html:513` | `var x=Math.tanh(inSig*0.92)-k*Math.tanh(s3*0.74);` | A **single, symmetric** tanh sits outside a plain 4× one-pole cascade (`s0..s3`). Real transistor/diode ladders saturate *per stage*, and diode ladders saturate *asymmetrically*. One global symmetric tanh gives smooth Moog-ish rounding, not 303 growl — it's the architectural reason the filter can't grow teeth even before the caps in #1/#2 are considered. |
| 4 | `index.html:463-464` | `cutCurve=0.075+0.925*Math.pow(cut,0.64); baseHz=30*Math.pow(245,cutCurve);` | A pre-baked "musical" warping of the cutoff pot laid on top of the raw exponential mapping. This is the second patch attempt at the same symptom (see `PATCH_NOTES.md` `Math.pow(cut,0.70)` / `0.58` rollback notes) — a curve fight instead of a filter fix. |
| 5 | `index.html:504` | `if(fc<24)fc=24; if(fc>fcMax)fc=fcMax;` with `fcMax=12000` | A fixed 12kHz ceiling on the cutoff frequency, independent of sample rate/Nyquist. Cuts off exactly the region (top-octave sweep, self-osc pitch above ~5-6kHz) where the TD‑3‑MO stays vocal. |
| 6 | `index.html:675` | `comp=ctx.createDynamicsCompressor(); threshold=-2dB, ratio=8, knee=8, attack=3ms, release=120ms` | An 8:1 master-bus compressor sitting after everything, squashing exactly the transient/dynamic bite (accent stabs, resonance peaks) that steps 1–4 are supposed to restore. It re-neutralizes the fix at the output stage. |
| 7 | `ACIDRATCHET_MF_v5_4_CORE_REFIT.html` | duplicate `makeVoice` with an older, *slightly less* capped version (`k>4.15`, `fc>6500`) | A second copy of the whole engine that has silently drifted from `index.html`. Two files means two places to "fix" resonance, which is how it got neutralized twice already. |

None of items 1, 2, 5, 6 are numerically necessary — they are taste decisions
that were tightened, not physical requirements. The one guard that **is**
legitimate and must stay is the NaN/blow-up trap at `index.html:520`
(`if(s3!==s3||s3>8||s3<-8){...}`) — that's a stability net for float
divergence, not a resonance neutralizer, and removing it would risk real
audio dropouts, not more bite.

---

## Phase 0 — Baseline capture

1. Render fixed test passes with the **current** engine before touching code:
   a single low note at reso 0/50/85/100%, cutoff sweep 0→100% over 8s at
   fixed reso 80%, and one full pattern from each factory preset
   (`ACID`, `SCREAM`, `MELT`, `DESTROY`, `INIT`).
2. Save as reference WAVs (use the existing WAV-export tap at
   `index.html:822-834`) and label them `baseline-*.wav`.
3. These become the "before" side of every later A/B comparison. Do not skip
   this — every later phase is validated against these files, not against
   memory of "it sounded thin."

## Phase 1 — Remove the resonance ceiling and the frequency-dependent damping

Target: `index.html:506-509`.

```js
// before
var k=resoK+sm.squelch*0.32+accSweep*0.10;
if(lfoTarget===3){ k+=lfoVal*lfoAmt*1.15; if(k<0)k=0; }
if(k>4.22)k=4.22;                                 // hard ceiling: bite, not whistle
if(fc>7200)k*=Math.sqrt(7200/fc);                  // keep high cutoff alive longer than previous 6500 rolloff
```

```js
// after
var k=resoK+sm.squelch*0.32+accSweep*0.10;
if(lfoTarget===3){ k+=lfoVal*lfoAmt*1.15; }
if(k<0)k=0;                                        // only sign guard remains
```

Delete the ceiling and the `fc>7200` damping outright. Do **not** replace
them with a softer version of the same idea (e.g. a higher ceiling, or a
damping curve that kicks in later) — that is the same mistake at a different
threshold. The self-limiting has to come from the nonlinearity inside the
loop (Phase 2), not from a value clamp outside it.

This step will make the filter unstable/screechy on its own — that's
expected and correct. It's the reason Phase 2 has to land before this is
listenable, and both phases should be committed together, not shipped
separately.

## Phase 2 — Replace the single global tanh with a per-stage, asymmetric ladder

Target: `index.html:512-519`, the oversampling loop.

```js
// before
for(var ov=0;ov<os;ov++){
  var x=Math.tanh(inSig*0.92)-k*Math.tanh(s3*0.74);
  s0+=g*(x  - s0);
  s1+=g*(s0 - s1);
  s2+=g*(s1 - s2);
  s3+=g*(s2 - s3);
  fout=s3;
}
```

Replace with a 4-stage cascade where **each stage** carries its own
saturation, and the saturation is **asymmetric** (positive/negative half
driven differently, the way a diode's forward-voltage asymmetry behaves),
instead of one symmetric tanh sitting outside the whole loop:

```js
// after
function diodeSat(x){
  // asymmetric soft clip: passes more level on the positive half than the
  // negative half, the source of the 303's even-harmonic "growl"
  return x>=0 ? Math.tanh(x*1.0) : Math.tanh(x*1.35)*0.82;
}
for(var ov=0;ov<os;ov++){
  var x=inSig - k*s3;                 // feedback tap stays linear; nonlinearity lives per-stage now
  s0+=g*(diodeSat(x)  - s0);
  s1+=g*(diodeSat(s0) - s1);
  s2+=g*(diodeSat(s1) - s2);
  s3+=g*(diodeSat(s2) - s3);
  fout=s3;
}
```

This is the actual architectural fix, not the cap in Phase 1. The reason
resonance can now go to full self-oscillation *without* screeching into
white noise is that each stage's own saturation gently compresses the loop
gain as amplitude rises — that's what a physical ladder does, and it's why
the TD‑3‑MO can sit at max resonance and produce a clean, controllable sine
rather than an exploding signal. Tune the `1.0` / `1.35` / `0.82` constants
by ear against Phase 0's baseline captures and against a TD‑3‑MO/TE‑3‑MO
recording (Phase 7) — do not tune them by re-adding a `Math.min`/`if(k>...)`
clamp if it still feels too hot. If it's too hot, the fix is in these
constants or in `resoK`'s scaling (`index.html:466`), not in a post-hoc
ceiling.

## Phase 3 — Remove the pre-baked cutoff curve and the fixed 12kHz ceiling

Target: `index.html:462-464` and `index.html:504`.

```js
// before
var cut=sm.cutoff;
var cutCurve=0.075+0.925*Math.pow(cut,0.64);
var baseHz=30*Math.pow(245,cutCurve);
...
var fcMax=12000;
...
if(fc<24)fc=24; if(fc>fcMax)fc=fcMax;
```

```js
// after
var baseHz=20*Math.pow(1000,sm.cutoff);            // honest 20 Hz .. 20 kHz exponential sweep, no taste-warping
...
var fcMax=sr*0.45;                                  // Nyquist-safe ceiling for numerical stability only
...
if(fc<20)fc=20; if(fc>fcMax)fc=fcMax;
```

The `0.075 + 0.925*pow(cut,0.64)` curve exists purely to compensate for the
fact that resonance was capped and the top of the range was damped (Phases 1
and 2 fix the actual cause). Once bite is restored by the filter topology
itself, a plain exponential pot mapping is enough — and it means the "sweet
spot" moves naturally with resonance/env/accent instead of being glued to
one fixed knob position. `fcMax` changes from an arbitrary genre-taste number
(12000) to a value tied to the actual sample rate, so it only exists to stop
the one-pole coefficient `g=1-exp(-2π·fc/(sr·os))` from wrapping past
Nyquist — a numerical requirement, not a musical one.

## Phase 4 — Loosen the master-bus compressor

Target: `index.html:675`.

```js
// before
comp.threshold.value=-2; comp.knee.value=8; comp.ratio.value=8; comp.attack.value=.003; comp.release.value=.12;
```

```js
// after
comp.threshold.value=-8; comp.knee.value=4; comp.ratio.value=2.5; comp.attack.value=.01; comp.release.value=.25;
```

An 8:1 ratio at a -2dB threshold with a 3ms attack squashes exactly the
accent stabs and resonance peaks that Phases 1–3 just spent effort restoring
— it's a limiter doing at the output what the filter cap was doing at the
source. This should function purely as a safety net against inter-sample
clipping when drums + bass sum, not as a de-facto second resonance cap.
Validate with Phase 0's accent-heavy pattern captures: peak reduction on
accented steps should be barely visible on the compressor's gain-reduction
meter, not a steady few dB.

## Phase 5 — Retire the duplicate engine file

`ACIDRATCHET_MF_v5_4_CORE_REFIT.html` carries its own copy of `makeVoice`
that has already drifted from `index.html` (`k>4.15`/`fc>6500` vs.
`k>4.22`/`fc>7200` — two independent attempts at the same clamp). Two
sources of truth are how "add another cap" happened twice already.

- Confirm `index.html` is the file actually deployed/served.
- Either delete `ACIDRATCHET_MF_v5_4_CORE_REFIT.html` from the repo, or turn
  it into a static, clearly-labeled snapshot (e.g. move to `/archive/`) that
  is never edited again.
- Do not hand-apply Phases 1–4 to both files independently — if the archive
  file is kept at all, it should be regenerated from `index.html`, not
  patched in parallel.

## Phase 6 — Re-audit accent/env headroom now that the cap is gone

Target: `index.html:444, 469-473, 503, 506` (`accCharge`, `accSweep`,
`accFiltDepth`, and `k=resoK+sm.squelch*0.32+accSweep*0.10`).

With the hard ceiling gone, `k` can now genuinely reach and exceed the old
4.22 boundary under accent stacking + squelch + high `reso`. That's the
intended new behavior (climbing accents should be able to push the filter
into self-oscillation "wow," matching the real TB‑303 accent-sweep
character) — but it needs to be re-listened, not re-capped:

- Sweep `accent`, `squelch`, and `reso` each to 100% individually and in
  combination; confirm the per-stage saturation from Phase 2 keeps the
  output bounded (watch the NaN/blowup guard at `index.html:520` — it should
  never trigger in normal playing; if it does, the saturation constants from
  Phase 2 need adjustment, not a new external clamp).
- Re-tune `accDrain`/`accLagTau` (`index.html:470-471`) only if the climbing
  accent now overshoots audibly — by ear, against Phase 0 baselines and
  Phase 7 hardware reference, not by capping `k`.

## Phase 7 — Calibrate against Behringer TD‑3‑MO / TE‑3‑MO

1. Record (or source) reference clips from a TD‑3‑MO or TE‑3‑MO: a resonance
   sweep to full self-oscillation on a static note, a cutoff sweep at high
   resonance, and one accent-heavy acid pattern.
2. Match test conditions as closely as possible to Phase 0's baseline
   captures (same note, same pattern, same knob sweep timing) and re-render
   with the new engine.
3. Compare in a spectrogram viewer (e.g. Audacity/iZotope RX) side-by-side:
   - Resonance sweep should show a continuous, unbroken rise in the resonant
     peak's Q all the way to a clean self-oscillating sine at max resonance
     — no ceiling/plateau in the spectrogram.
   - Cutoff sweep at high resonance should keep harmonic energy alive at the
     top of the range, not taper off before 100%.
   - Accent pattern should show visible transient peaks in the loudness
     envelope, not a flattened compressor-glued waveform.
4. Iterate Phase 2's saturation constants and Phase 3's `baseHz` mapping
   constant (currently `20 * 1000^cutoff`) until the spectrogram shape and
   the by-ear character match the hardware reference. This is the only phase
   where knob-feel constants should still move — everything structural
   (caps, dampers, the pre-baked curve, the compressor) is already gone by
   this point.

## Phase 8 — Preset re-audit

Target: `index.html:904-908` (factory presets `ACID/SCREAM/MELT/DESTROY/INIT`)
and `index.html:1083-1101` (the extended preset bank).

Every stored preset's `reso`/`cutoff`/`env`/`accent` values were tuned by
ear against the old capped filter. After Phases 1–4 land, replay each preset
against its own entry in Phase 0's baseline set and adjust only if the
preset now sounds broken (e.g. constant self-oscillation where the original
intent was a controlled squelch) — most presets should simply sound more
alive with no numeric changes needed, since the knob values themselves
didn't move, only what they drive.

## Explicit non-goals

- Do not reintroduce any `if(k > <number>) k = <number>` style ceiling on
  resonance, in any form, at any threshold.
- Do not reintroduce any cutoff-frequency-dependent resonance damping
  (`k *= sqrt(X/fc)` or equivalent) — resonance behavior must be uniform
  across the cutoff range, exactly as it is on the TD‑3‑MO/TE‑3‑MO.
  Any two-cent knob-feel patch is more of the same mistake and should be rejected in review.
- Do not compensate by tightening the master compressor (Phase 4 goes one
  direction only: looser).
- Do not add a second "musical curve" on top of the cutoff mapping to
  compensate for lost top-end energy — if the top end is dead after Phase 2,
  that's a saturation-tuning problem, not a mapping-curve problem.

## Acceptance criteria (definition of done)

1. Resonance knob at 100% with `cutoff` static and no note re-trigger
   produces continuous, stable self-oscillation (audible pure tone tracking
   `baseHz`), not a hard-clipped or capped tone.
2. A cutoff sweep from 0→100% at fixed reso 80% shows no audible dip or
   "damping zone" anywhere in the sweep, including the top octave.
3. `index.html` has zero remaining unconditional numeric clamps on `k`
   besides the `k<0` sign guard.
4. The master compressor's gain-reduction meter shows near-zero reduction
   during normal accented playing, only engaging on genuine peak/summing
   events.
5. Spectrogram comparison against TD‑3‑MO/TE‑3‑MO reference clips (Phase 7)
   shows matching resonance-sweep and cutoff-sweep shapes, not just similar
   knob ranges.
6. Only one engine source file remains authoritative (Phase 5).
