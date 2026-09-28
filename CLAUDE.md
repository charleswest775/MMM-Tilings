# MMM-Tilings — context for Claude sessions

Charles's own MagicMirror² module: tilings that never repeat, for his hallway mirror, laid tile
by tile from the centre, then held. One page in a rotation of pages.

Split out of MMM-ChaosTheory on 2026-09-28, with its history (it was the `tilings` page there).
The shell (`MMM-Tilings.js`, the `node_helper.js` stats panel, `dev/preview.html`) is shared in
spirit with the sibling modules (MMM-ChaosTheory, MMM-Atom, MMM-FractalZoom, MMM-Chladni,
MMM-SacredGeometry, MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck, all under
~/dev/mirror-modules or ~/dev): a fix there probably belongs in the siblings too.

## What exists (v1.0.0)

- `MMM-Tilings.js` — module shell: one canvas plus an HTML caption (updated 2×/s). Moves to a
  new tiling every `cycleSeconds` and on each `resume()`. Loop: `setTimeout` until a frame is
  due, then one `requestAnimationFrame`. `suspend()` stops it and clears the canvas; a sim with
  `resting = true` is polled only every 500 ms. While MagicMirror fades the module out (`hidden`
  is set at the start, `suspend()` comes after), frames draw nothing. `turns: { of: n, at: k }`
  lets modules on one MMM-pages page take turns: each counts its showings and shows only on its
  own (wrapper `display: none` and no loop otherwise). On the mirror it shares the page with
  MMM-SacredGeometry and MMM-PlanetsDance, turns of 3 at 0, 1, 2 (this one at 1).
- Simulations are classes on `window.TilingsSimulations` (here only `tilings`) with `step(dt)`,
  `draw(ctx, w, h)`, optional `readout()` and `info`. UMD-style so it runs in Node.
- `simulations/tilings.js`: a deck of ten kinds (and a deck of five palettes): Penrose,
  Ammann–Beenker, heptagonal, dodecagonal (de Bruijn multigrid, random offsets; Penrose's sum to
  0), five hyperbolic {p,q} in Poincaré's disc (reflections, geodesic arcs, Coxeter's shaded
  triangles), and the hat. Undocumented constructor options, read from the config: `kind`
  (a key, e.g. `"hat"`, `"hyperbolic-7-3"`) and `seed`.
- `simulations/hat.js` (`window.TilingsHat`): the paper's H/T/P/F metatile substitution after
  Kaplan's reference code; tests check no overlaps/gaps and 4, 25, 169, 1156 hats per level.
- Laid in a spiral in `tilingsSeconds` (22), then rests: ~1% of the canvas changes per frame.
- `tests/` — `node --test`, no dependencies: multigrid tilings without overlaps or gaps, φ and
  √2 ratios, hyperbolic congruence, the hat's counts, congruence and φ⁴ ratio, every kind builds.
- `dev/preview.html` runs the module in a desktop browser (serve the module folder, e.g.
  `python3 -m http.server`; `?kind=hat`); the stats panel gets made-up numbers there.

## Measured cost on the Pi

700², 12 fps, seven showings, Electron + cage: ~33% of a core while laying (mostly the
per-frame fixed cost), ~7% held, 37% (34–39) over a 30 s showing, laid by ~23 s. Hidden: 0.3%
(baseline 0.2%).

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally, keep each frame's changes spatially compact (hence the spiral), and
  rest when the picture is static. Line width, opacity, `rAF` vs timer made no difference.
- Use `statsPanel: true` or `debugStats: true` and a `grim` screenshot to see fps on the Pi.
  MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be
  set that way.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. A full-screen 60 fps canvas could cost a lot more.
  **Measure, don't guess**: on the Pi, `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS
  over 60 s. Record before/after numbers in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). Verify that
  `suspend()`/`resume()` actually fire on page changes. If the loop keeps running while
  hidden, it burns CPU 24/7.

## Deploying and testing

- This repo is public (github.com/charleswest775/MMM-Tilings) so the Pi can `git clone`/`git pull`
  without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-Tilings`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: `dev/preview.html` or MagicMirror on the Mac in a browser, then confirm
  performance on the Pi.
- Commit as Charles's GitHub noreply address (see git config in this repo).
