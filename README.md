# ACIDRATCHET MF79

A single-file, browser-based **TB-303 acid synthesizer** — no install, no account, no cloud. Open the HTML file and it runs.

## What it is

A digital recreation of the classic 303 acid bassline machine, built as one self-contained HTML file with a real-time **AudioWorklet** DSP engine. It runs equally as a **web tool** (deploy anywhere, e.g. Netlify) or as a **standalone offline app** (save the file, open it locally — works without a connection).

## Highlights / USPs

- **Single file, zero dependencies.** One `.html` — no build step, no libraries, no server. Fully offline-capable.
- **Authentic 303 DSP, not a clone-by-numbers.** Diode-style ladder filter with genuine self-oscillation, tuned to sit on the musical "knack" just below the whistle.
- **Real accent circuit.** Models the accumulating capacitor behavior — consecutive accents climb (the classic acid screw-up), sweeping the filter, not the pitch.
- **AudioWorklet engine.** Sample-accurate, low-latency, runs off the main thread.
- **Mobile-first.** Built and voiced by ear on iPhone/iPad — touch UI, works on desktop too.
- **Sequencer + drums.** Pattern sequencer with slides/accents, song chaining, drum kits (CORE / 808 / 909), and a preset system.

## Usage

**Web:** drop the HTML on any static host (Netlify, GitHub Pages, etc.) and open the URL.
**Standalone:** download the file, open it in a modern browser. AudioWorklet needs `https://` or `localhost` — a plain `file://` open falls back to a simpler audio path.

## License — Non-Commercial, Revocable

Copyright © 2026 **Manfred Foissner** (aka *M4n@R4TCh3T*). All rights reserved.

Permission is granted to **use, run, and study** this software for **personal, non-commercial purposes only**, subject to:

1. **No commercial use** of any kind without prior written permission from the author.
2. **No redistribution, resale, or sublicensing** of the software or modified versions.
3. This permission is granted **until revoked** — the author may withdraw it at any time.
4. This notice and copyright must remain intact in all copies.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. THE AUTHOR IS NOT LIABLE FOR ANY CLAIM, DAMAGE, OR OTHER LIABILITY ARISING FROM ITS USE.

## Contact

For commercial use, permissions, or inquiries: **vtr1979@gmx.at**
