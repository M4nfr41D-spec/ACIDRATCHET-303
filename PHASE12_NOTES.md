# ACIDRATCHET-303 FILTER BITE PHASE 1-2

## Scope

This is an experimental architecture patch based on `ROADMAP.md`.

Implemented only:

1. Remove external resonance ceiling and high-frequency resonance damping.
2. Replace the single global symmetric feedback tanh with per-stage asymmetric saturation.

Not implemented yet:

- Phase 3 cutoff remap / fcMax change
- Phase 4 compressor change
- Phase 5 duplicate file cleanup
- Preset retuning

## Exact behavioural change

### Removed

```js
if(k>4.15)k=4.15;
if(fc>6500)k*=Math.sqrt(6500/fc);
```

Only the sign guard remains:

```js
if(k<0)k=0;
```

### Added

```js
function diodeSat(x){
  return x>=0 ? Math.tanh(x*1.0) : Math.tanh(x*1.35)*0.82;
}
```

### Filter loop changed

From one global tanh:

```js
var x=Math.tanh(inSig*0.92)-k*Math.tanh(s3*0.74);
```

To per-stage asymmetric saturation:

```js
var x=inSig-k*s3;
s0+=g*(diodeSat(x)  - s0);
s1+=g*(diodeSat(s0) - s1);
s2+=g*(diodeSat(s1) - s2);
s3+=g*(diodeSat(s2) - s3);
```

## Test instruction

Start conservative. Do not judge this with all knobs at 100% first.

Suggested first pass:

- Saw
- Resonance 65%
- Cutoff 45–80% sweep
- Env 50–70%
- Decay 40–60%
- Accent medium

Then test full resonance.

## Expected outcome

More upper-range bite and self-oscillation potential. This may also expose unstable zones. If it screams too much, the next correction should tune `diodeSat()` constants or `resoK`, not re-add an external k cap.
