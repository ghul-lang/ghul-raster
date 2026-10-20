# Cloud code review brief

What this repository is, and what to watch for in it. Everything else - what PR
context is available, how to post a review, what makes a finding worth raising,
comment hygiene, PR-description shape, the versioning mechanism - comes from the
review workflow's runtime notes. Don't restate it here: this file is read first,
so a stale copy would silently override the current text.

Not loaded by local Claude Code; only the cloud reviewer reads this.

## What this repo is

`ghul-raster` is a ghūl library, published as the `ghul.raster` NuGet package,
that draws into an in-memory image and writes it as a PNG. It is managed code
throughout, with no dependency on a graphics stack, because it has to run
wherever ghūl does: on Linux, where `System.Drawing.Common` throws, and under
WebAssembly in the ghūl playground, where a native library cannot load.

`README.md` is the documentation and is packed into the package, so it is the
contract: coordinates and `view`, strokes, shapes, pixels, reading a PNG, the
turtle, text in a Hershey stroke font, axes and colour maps, and the
`<<image name.png>>` line `show` prints so that whatever displays a program's
output knows where a picture belongs.

Its consumers assert images byte for byte. `ghul-rosetta-code` keeps a
`*.png.expected` beside each solution that draws, and those pictures are
published on Rosetta Code; the playground compiles programs against this
package and ships the same assembly to the browser.

## What to watch for here

- **A change to what existing calls draw.** Exact pixel output is not a
  compatibility promise. A fix or an improvement may change the pixels the same
  sequence of calls produces - antialiasing, rounding, stroke ends, fill
  coverage, text metrics, a GIF palette - when the description says so and the
  difference is the evident consequence of the change. Flag a pixel change that
  is an unwanted side effect or an unrelated regression: one the description
  does not account for, or that moves pictures the change should not touch. An
  expected image that moves should move for the stated reason and no other.
- **Deterministic output.** The PNG encoder must produce the same bytes for the
  same image on every platform and run: no timestamps or optional chunks, no
  dependence on hash ordering, culture or floating-point formatting, and no
  compression setting that varies with the runtime.
- **Nothing the browser cannot run.** No native dependency, no P/Invoke, no
  threads, no API missing from the wasm runtime. The filesystem is only
  touched by `write` and `read`, with the path the caller gave.
- **User coordinates against pixels.** Which calls take `view` coordinates and
  which take pixels is documented per call, and a new primitive should follow
  the nearest existing one. A shape that ignores the view's y direction, or
  stretches differently from `line`, is a bug a test image may not show.
- **Out-of-range input.** Positions outside the image are ignored by the
  writers rather than thrown on; a new primitive that indexes the buffer
  directly needs the same clipping.
- **The README stays true.** A change in behaviour with no matching change to
  the README is a finding, and so is README text describing something the
  diff does not do.
- **Tests carry an image.** A new primitive wants an integration test whose
  expected PNG a person has looked at, not only a unit test of its arithmetic.

## Versioning

Major means a consumer's existing program stops compiling, or the
`<<image ...>>` contract changes, or an unwanted side effect changes what it
draws. Different pixels that are the described, expected consequence of a fix or
an improvement do not make a change major. Minor means additions: new
primitives, new overloads, new helpers, and a fix or improvement whose expected
consequence is different pixels from the same calls, since a consumer's expected
images move at the next bump. Patch is a fix that draws the same pixels for
correct programs, or changes nothing a program can observe.
