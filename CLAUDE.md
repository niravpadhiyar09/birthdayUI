# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page "birthday film" — one HTML page that plays a cinematic sequence in
four acts (draw a Cupid bow → strike a heart → rose flood + kinetic wish → a
canvas tree that blooms into a heart of petals). Vanilla JS + GSAP, one `<canvas>`,
no UI framework. Not an app; a self-contained animated greeting.

## Commands

```bash
npm run dev      # Vite dev server (hot reload)
npm run build    # bundle to dist/ (gsap gets bundled here — see Deploy gotcha)
npm run preview  # serve the built dist/ locally
```

No test suite, no linter. "Verifying" a change means running `npm run dev` and
watching the film — ideally the whole sequence, since acts hand off to each other.

## Architecture

Three source files drive everything: [index.html](index.html),
[birthday.css](birthday.css), [birthday.js](birthday.js). `dist/` is generated —
never edit it by hand.

**The four acts and how they hand off** (all narrated in the header comment of
[birthday.js](birthday.js)):

- **Acts 1–3 are GSAP**, one master timeline built in `buildFilm(m)`. Act 1 is the
  interactive bow (pointer-drag or keyboard on `#archery`); firing (`fire()`)
  starts the timeline. Act 2 = arrow strike + rose `#flood`. Act 3 = kinetic
  headline masked/hinged out of that colour. No cross-fades anywhere — this is a
  deliberate style rule; colour *arrives* (via the flood circle / masks), it never
  opacity-fades between acts. Preserve that.
- **Act 4 is raw canvas 2D**, its own `requestAnimationFrame` loop (`treeFrame`),
  started by the GSAP timeline's end. The tree is procedurally grown
  (`buildScene` / `addBranch`) constrained inside a heart polygon (`buildHeartPoly`
  / `pointInPoly`), then blossoms rise and settle. It plays **once and holds** —
  living, never looping. `window.bdayDone` flips true when it settles; `#replay`
  arms ~1s later and `resetAll()` rebuilds the whole film.

**Coordinate coupling (fragile):** the arrow must physically land on the heart.
`refreshRig()` / `shotGeom()` read live `getBoundingClientRect()` of the bow,
string, arrow tip (`#tip`), and target to compute draw geometry and flight path in
real pixels. Changing the bow/arrow SVG sizes or their CSS positioning will throw
off the landing — re-check `refreshRig` after any such edit. Everything re-measures
on `resize`.

**Accessibility / reduced motion:** `reduceMotion` (media query) and the `.sr-only`
line are first-class. Decorative nodes are `aria-hidden`; the two real controls are
the heart `<button>` and the `role="button"` `#archery` rig. Keep new decorative
elements out of the a11y tree.

## Record mode (how media/demo-silent.mp4 is captured)

Loading with `?record` in the URL (`isRecord`) exposes `window.bdayAPI`,
`window.bdayCues`, and `window.bdayDone`, and records cue timestamps (`cue(name)`)
against the soundtrack's t=0 (the moment of draw). This is the hook an external
screen-recorder script drives to produce [media/demo-silent.mp4](media/demo-silent.mp4).
Don't remove the `?record` plumbing when refactoring the timeline.

## Personalization (URL params — no form)

**Opening the page plays the film straight away.** There is no sender/builder UI —
the husband just sends the link, the recipient opens it, the film runs. The card
is a **from Nivid → to Nisha** default, baked into `DEFAULT_TO/FROM/MSG` in
birthday.js. Any of it can be overridden via the URL query (parsed with the same
`URLSearchParams` the recorder uses):

- `?to=Nisha` — recipient name → "for Nisha" (Act 1 eyebrow) + "Happy Birthday,
  Nisha" (`#wHero`, Act 4). Defaults to `Nisha`.
- `?from=Nivid` — signs a "with love, Nivid" line (`#wFrom`). Defaults to `Nivid`.
- `?msg=...` — the closing subline (`#wSub`). Defaults to the romantic line.

`applyCard(card)` writes these with **`textContent` (never `innerHTML`)** so a name
can't inject markup, and runs *before* any reveal so the existing fades/hinges
animate the personalized copy — the headline glyph-split (`splitWord`) is left
alone, name goes in the hero to avoid 2-line layout breakage.

Boot always calls `enter()` (the film). `?pick=1` jumps straight to the finale
(dev preview); `?record` is the recorder path. The end-of-film **Share** button
(`#share`) copies the current URL via `copyText` (clipboard, with an address-bar
fallback) so the link can be re-shared.

## Photos of the birthday person

Cut-out photos (transparent-background PNGs) bloom among the tree's branches in
the finale. Drop files in `photos/` — any `.png/.jpg/.jpeg/.webp` is picked up by
`import.meta.glob('./photos/*…', { as:'url' })` at build time, sorted by name,
first four used. `photos/originals/` holds the full-res source backups; the
working copies are downscaled to ≤1600px (keep them web-sized — full-res phone
PNGs are 5–6 MB each and tank load time). No photos present → numbered SVG
placeholders render so the layout still previews.

`buildPhotos()` creates the `.photo` figures in `#photos`; `setPhotos(on)` is
called from `showWish()` so they reveal/hide exactly with the wish and sway
gently (`photoSway`). Positions (`.photo--0..3` in birthday.css) are kept clear of
the wish, which anchors lower-left (`.wish{ left:5.5%; bottom:10% }`): one photo
upper-left above the wish, three down the right edge. If you move the wish or add
photos, re-check overlap. `?photos=off` disables photos; `?pick=1` jumps straight
to the finale (a fast preview, used during development).

## Deploy

GitHub Actions ([.github/workflows/deploy.yml](.github/workflows/deploy.yml)) runs
`npm ci && npm run build` and publishes **`dist/`** to Pages on push to `main`.
Critical: it must publish the *built* output, not raw source — serving source made
the browser hit un-bundled `import 'gsap'` and the film never ran. If you touch the
deploy path, keep the build step.

**Base-path gotcha:** [vite.config.js](vite.config.js) sets `base: '/'` (root),
while the deploy.yml comment references a `/happy-birthday-tree/` subpath. If this
ever moves to a *project* Pages URL (`user.github.io/repo/`), `base` must change to
`'/repo/'` or all asset URLs 404. Root `base` only works for a user/org root Pages
site or a custom domain.
