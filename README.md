# TypeEffects

Three small browser demos that animate how text appears on screen: a top-down
wipe, a self-drawing outline, and a spray-paint effect with overspray and drips.

Each demo is one self-contained HTML file. There is no build step and nothing
to install. Open a file in any browser on macOS, Linux, or Windows.

| File | Effect | Technique |
| --- | --- | --- |
| `wipe.html` | Text is revealed from the top down | CSS `clip-path` animation |
| `trace.html` | Each letter's outline draws itself, then fills in | SVG `stroke-dasharray` / `stroke-dashoffset` |
| `spray.html` | Letters are spray-painted in rows, with overspray and drips | Canvas 2D, a pixel stencil, and random dots |

## Background: why not change the font renderer?

The operating system draws text with its own rasterizer: Core Text on macOS,
FreeType on Linux. The rasterizer turns a glyph's vector outline into pixels in
one step, and it offers no partially drawn state you could animate.

To animate the drawing of text, you have two choices:

1. **Let the system draw the text, and control which part is visible.** The
   wipe uses this approach.
2. **Get the glyph shapes and draw them yourself.** The trace uses SVG outlines
   that the browser builds from the font. The spray paints dots itself, using a
   pixel copy of the word as a stencil.

Both approaches are cross-platform because the browser handles the font.

## `wipe.html`: Top-down wipe

The word is ordinary HTML text. The animation changes only how much of the text
is visible.

```css
.word     { clip-path: inset(0 0 100% 0); }   /* clipped away completely */
.word.run { animation: wipe 1.6s forwards; }
@keyframes wipe { to { clip-path: inset(0 0 0 0); } }
```

`inset(top right bottom left)` describes a rectangle measured in from each edge
of the element. A bottom inset of `100%` leaves a rectangle with no height, so
nothing shows. Animating the bottom inset to `0` moves the rectangle's lower
edge down the text, which reveals the letters row by row.

**Try:** animate the `left` value instead for a left-to-right reveal, or use
`circle()` for a reveal that grows from the center.

## `trace.html`: Stroke trace

The word is an SVG `<text>` element. It starts with no fill and a colored
outline (`stroke`), and the outline is hidden by a dash trick:

```css
.word {
  fill: transparent;
  stroke-dasharray: 700;    /* one dash 700 units long, then a 700-unit gap */
  stroke-dashoffset: 700;   /* shift the pattern so the gap covers the outline */
}
@keyframes draw { to { stroke-dashoffset: 0; } }
```

As the offset goes from 700 to 0, the dash slides along each glyph's outline,
and the outline appears to draw itself. A second animation, `fill-in`, starts
just before the drawing ends and fades the fill to the stroke color.

The dash length only needs to be longer than the longest single glyph outline.
700 works for Bungee at 140px. A larger font, or a face with more detailed
outlines, may need a bigger value.

**Limitation:** every glyph uses the same dash length, so short outlines finish
drawing before long ones. For exact timing, convert each glyph to a `<path>`
(for example with [opentype.js](https://opentype.js.org/) and a local `.ttf`
file), then call `path.getTotalLength()` to set each glyph's dash length.

## `spray.html`: Spray paint

This demo draws every dot itself on a `<canvas>`. The work happens in four
parts.

### 1. Build a stencil

`buildStencil()` draws the word once, in solid color, on a hidden canvas, then
reads the canvas pixels with `getImageData()`. `isInside(x, y)` checks the
alpha (opacity) value of a pixel and reports whether that point is inside a
letter. The stencil also records the word's bounding box, which tells the
nozzle where to sweep.

This method works with any font the browser can draw, and it never reads the
font file.

### 2. Sweep the nozzle

An imaginary nozzle sweeps back and forth across the bounding box, moving down
one row at the end of each pass, from top to bottom. On each animation frame it
moves `speed` pixels and sprays along the stretch it just covered.

### 3. Spray dots with a Gaussian spread

Real spray paint is densest at the center of the cone and fades toward the
edges. `gaussian()` uses the Box–Muller transform to turn two uniform random
numbers into one normally distributed number, and each dot is offset from the
nozzle by `gaussian() * spread` in both x and y.

Each dot is then tested against the stencil:

- **Inside a letter:** painted at 50% opacity. Overlapping dots build up toward
  solid color, and that build-up gives the grainy spray texture.
- **Outside a letter:** usually discarded. About 4% are painted at 10% opacity,
  as overspray.

### 4. Add drips

A drip can start only when a painted dot sits at a letter's bottom edge, where
the point `spread` pixels below it is outside the stencil. Even then, it starts
only 2% of the time, and at most 14 drips are kept.

Each drip grows downward a little every frame. Its step size shrinks as it
nears its target length, so it slows down like a real run of paint. A small
dot is painted at the end of the drip.

### Settings to change

Every setting scales with the font size, so the effect looks the same at any
canvas size.

| Variable | Effect of increasing it |
| --- | --- |
| `spread` | A wider, softer spray cone |
| `rowGap` | More space between passes, giving a streakier result |
| `speed` | Faster sweeps with lighter coverage per pass |
| `dotsPerFrame` | Denser paint |
| Drip chance (`0.02`) and limit (`14`) | More drips |

Change `WORD`, `FONT`, and `PAINT` at the top of the script to change the
text, the typeface, and the color. Any font used in `FONT` must also be loaded
in the Google Fonts `<link>`. The script waits for the font to load, because
otherwise the stencil would be drawn in a fallback typeface.

## Shared details

- **Replay:** the CSS demos restart their animations by removing the `run`
  class, forcing a layout with `offsetWidth` or `getBoundingClientRect()`, and
  then adding the class back. Without the forced layout, the browser would
  merge the remove and the add, and nothing would restart. The spray demo
  replays by cancelling its animation frame and starting again.
- **Fonts:** all three demos use [Bungee](https://fonts.google.com/specimen/Bungee)
  from Google Fonts. The animation starts only after `document.fonts.load()`
  finishes.
- **Accessibility:** when the system's reduced-motion setting is on
  (`prefers-reduced-motion: reduce`), each demo skips the animation and shows
  the finished word.
- **Sharp canvas:** `spray.html` sizes the canvas buffer by `devicePixelRatio`
  so the dots stay sharp on Retina and other high-DPI screens.

## Ideas for next steps

- Trace each glyph precisely with opentype.js and `getTotalLength()`.
- Move the nozzle along each letter's centerline instead of in rows, so the
  word is painted the way a person would write it.
- Port the spray effect to Python with `fontTools` or `freetype-py` and
  `pycairo`, write out the frames, and join them into a video with `ffmpeg`.
- Move the scripts into `.js` files so ESLint and Jest can run on them, for
  example to test `gaussian()` and `isInside()`.
