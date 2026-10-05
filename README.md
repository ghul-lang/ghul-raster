# ghul-raster

[![CI](https://img.shields.io/github/actions/workflow/status/degory/ghul-raster/cicd.yml?branch=main)](https://github.com/degory/ghul-raster/actions/workflows/cicd.yml?query=branch%3Amain)
[![NuGet version (ghul.raster)](https://img.shields.io/nuget/v/ghul.raster.svg)](https://www.nuget.org/packages/ghul.raster/)
[![License](https://img.shields.io/github/license/degory/ghul-raster)](https://github.com/degory/ghul-raster/blob/main/LICENSE)
[![ghūl](https://img.shields.io/badge/gh%C5%ABl-100%25!-information)](https://ghul.dev)

Drawing into a PNG, from ghūl, with no dependency on a graphics stack.

`System.Drawing.Common` is GDI+ and throws on Linux; SkiaSharp needs a native
library; neither runs under WebAssembly. This library allocates a buffer,
draws strokes into it, and writes a PNG, all in managed code, so it runs
wherever ghūl does.

```ghul
use Raster.IMAGE

let image = IMAGE(640, 400)

image.view(-0.5D, -1.4D, 6.8D, 1.4D)

image.colour(20ub, 60ub, 200ub)
image.stroke(2.0D)

image.line(x0, System.Math.sin(x0), x1, System.Math.sin(x1))

image.text(0.1D, 1.15D, 18.0D, "sin x")

image.write("plot.png")
```

## Coordinates

`view` maps a rectangle of your own coordinates onto the whole image, with y
increasing upwards as it does in mathematics rather than downwards as it does
in an image. A program then works in the space its problem is stated in - the
complex plane, a unit square, the range its data covers - and says nothing
about pixels.

The rectangle is mapped exactly, so it stretches where its proportions differ
from the image's: a circle drawn in a square rectangle on a 640 by 320 image
comes out an ellipse. Choose proportions that match, or accept the stretch.

Without a `view`, positions are already pixels.

Given a fifth number, `view` maps the rectangle onto the image less that many
pixels on every side, which leaves room outside it for axes. `bounds` is the
rectangle the last `view` was given, and `to_user` undoes `to_pixel`.

Sizes are always in pixels, whatever the view. A label is as tall as it is
asked to be and a stroke as wide, because a label that stretched with the
axes would be unreadable on any plot whose axes differ in scale.

## Strokes

Everything is drawn as a stroke, and coverage is computed from the distance to
the line segment. Ends and joins are round, and the edges are antialiased.
`stroke` sets the width in pixels; `colour` sets what is laid down.

## Shapes

`line` takes a sequence of points as well as two, and draws through each in
turn with the joins rounded the way a single stroke's ends are. It leaves the
shape open, so to close one, repeat the first point.

`fill` takes the same sequence and fills the polygon it encloses, joining the
last point to the first. Edges are antialiased by sampling within each row, and
a boundary crossed twice leaves a hole rather than a second layer.

`circle`, `disc` and `arc` take a centre and a radius in user coordinates.
`disc` is filled; the other two are strokes, antialiased the same way. An arc
goes counter-clockwise from its first angle to its second, in degrees from the
x axis, so 270 to 90 is the right half. A view that scales x and y differently
draws each of them as an ellipse, as it would any other shape.

`dot` and `ring` mark a point in user coordinates with a radius in pixels, so
they stay round on a view that scales x and y differently: the marker for a
point on a plot. `fill_rectangle` fills between two corners.

Shapes filled one at a time each only partly cover the pixels along an edge
they share, so the background shows through as a pale seam between them.
`SHAPES` collects polygons, each with its own colour, and `fill` given a
`SHAPES` fills them together, adding up what each covers of a pixel first, so
a tiling or a row of bars meets without a seam. Where shapes overlap, their
colours mix.

That mixing is why a surface drawn as depth-sorted quadrilaterals cannot use
`SHAPES`: there the later shape has to cover the earlier one rather than blend
with it. Fill those one at a time, and close the seam by stroking each shape's
own outline in its own colour straight after filling it.

`flood_fill` recolours every pixel joined to the one it is given and the colour
that one is. It takes pixels rather than user coordinates, as `set_pixel` does:
the region is a property of the image rather than of what the view describes.

## Colour maps

`COLOUR_MAP` turns a number from 0 to 1 into a colour, and `colour` accepts what
it gives back, so a heat map is `image.colour(COLOUR_MAP.viridis(value))` for
each cell. `viridis` runs dark to light for a value that only grows;
`cool_warm` runs blue through grey to red for one either side of a middle at
0.5; `hue` goes once round the colour wheel for one that wraps, and wraps with
it. The first two clamp a value beyond either end.

## Axes

`AXES` draws a frame round the rectangle the view was given, with ticks at round
numbers along the bottom and left, each labelled, and a title for either axis if
`x_title` or `y_title` is set. It draws outside the rectangle, in the current
colour and stroke, so give the view a margin:

```ghul
image.view(0.0D, -1.0D, 10.0D, 6.0D, 50.0D)

let axes = AXES(image)

axes.x_title = "n"
axes.draw()
```

`fit` sets the view itself, with each margin sized to what `draw` will put
there, so a long or negative label is never cut off:

```ghul
let axes = AXES(image)

axes.y_title = "height"
axes.fit(0.0D, -12500.0D, 10.0D, 2500.0D)
axes.draw()
```

`x_ticks` and `y_ticks` put ticks at given values with given labels in place
of round numbers, one for each category of a bar chart, say. `key` adds a row
to a key in the top right corner inside the frame: a swatch of a colour and a
label.

Text widths are in pixels; `to_user` turns a pixel position into user
coordinates for placing anything else by them.

The scales are linear.

## Pixels

`clear` sets every pixel back to white, so a program drawing frame after frame
can reuse one image rather than allocating another for each.

`clear` can also paint every pixel one colour, for a background. `set_pixel`
with no colour writes the current one.

`set_pixel` writes a pixel outright and `pixel` reads one back. `blend` lays
part of the current colour over what is already there, which is what a program
drawing its own shapes needs to antialias them. A position outside the image is
ignored by both writers.

## Writing a PNG

`write` puts the image in a file, and `encode` gives the same bytes to a program
with somewhere else to send them. The output is deterministic: the same pixels
always encode to the same bytes.

## Animation

`ANIMATION` collects frames and writes them as one animated PNG, which a browser
plays and loops by itself. `add` copies an image as it stands, with how long to
show it in milliseconds, so a program can clear and redraw the same image for
each frame:

```ghul
let image = IMAGE(160, 120)
let animation = ANIMATION(160, 120)

for index in 0..12 do
    image.clear()
    draw(image, index)
    animation.add(image, 80)
od

animation.write("swing.png")
animation.show("swing.png")
```

It loops forever unless `loops` says how many times to play. Every frame is
the whole image, compressed on its own when it is added, and the output is
deterministic. A reader that does not know about animation, `IMAGE.read`
among them, sees the first frame.

`ANIMATION.read` and `ANIMATION.decode` read every frame back, with `frame`
giving each as an image and `delay` how long it shows. Each frame is drawn onto
what the ones before it left, as a browser would show it, so an animation that
updates part of the image reads correctly too. A PNG with no animation reads as
one frame.

### As a GIF

`write_gif` and `encode_gif` write the same frames as an animated GIF, for
somewhere that plays a GIF but not an animated PNG. A GIF holds at most 256
colours, shared here by every frame: if the frames use no more than 256
between them the GIF is exact, and otherwise 256 are chosen by median cut and
each pixel takes the nearest, without dithering. A GIF counts delays in
hundredths of a second, so they are rounded to the nearest hundredth, and
browsers show anything under two hundredths for a tenth, so a shorter delay
becomes two hundredths. `loops` carries over as the looping extension, which
counts repeats after the first play; an animation that plays once leaves it
out. The output is deterministic.

## Reading a PNG

`IMAGE.read` decodes a PNG file into an image, and `IMAGE.decode` does the
same from bytes already in hand. Eight bits a channel and not interlaced,
which is what anything writing a PNG for a program to read produces: grey,
truecolour and indexed, each with or without alpha. An image carries no alpha
of its own, so a transparent pixel is composited onto the white it starts as.

## A turtle

`TURTLE` is a pen on an image, at a position and a heading, told how far to go
rather than where to go: `forward`, `turn`, `face`, `move_to`, `pen_up` and
`pen_down`. A curve stated as a sequence of turns and steps is written that way
directly.

Angles are in degrees and follow the image's coordinates, so with a `view` set
a positive turn is counter-clockwise and without one it is the other way about.

## Text

`text` draws with its baseline at the position given and the height in pixels
from baseline to the top of a capital. An `Anchor` says whether the text starts,
is centred, or ends at its position, and `upward_text` reads bottom to top, as a
label beside a vertical axis does. `text_width` says how wide that text
will be, for placing it. The weight follows the height unless it is given.

The face is a Hershey stroke font, so text is drawn with the same strokes as
everything else, and scales without a rasteriser.

## Saying where an image belongs

A program that draws has two outputs, and nothing in its text says where the
pictures go.

```ghul
image.write("plot.png")
image.show("plot.png")
```

`show` writes a line naming the image:

```
<<image plot.png>>
```

That is a contract with whatever displays the output, not an instruction to
anything in particular. A publisher can turn it into a reference its
destination understands and put the file where that reference points; a
console that can display images can display it in place; a terminal shows the
line, which names a file the reader was told about anyway.

Nothing here knows where the output is going, and nothing that reads the
marker needs to know how the image was drawn.

A host that shows values - a notebook, a REPL - needs no file at all. An
`IMAGE` implements `Ghul.Renderable`, offering itself as `image/png`, so a
cell that ends on an image, or passes one to `display`, shows the picture
where the host honours that type:

```ghul
let image = IMAGE(200, 100)

image.line(10.0, 10.0, 190.0, 90.0)

display(image)
```

A host that does not honour it shows the image's text instead, as it would
for any other value.


## The font

The Hershey Fonts were originally created by Dr. A. V. Hershey while working
at the U. S. National Bureau of Standards. The format of the font data was
originally created by James Hurt, Cognition, Inc.

`fonts/futural.jhf` is the face and `fonts/hershey.txt` is the notice its
distribution terms require to accompany it. `build/embed-font.sh` writes the
face into `src/font-data.ghul`, which is what the library reads: the compiler
emits no embedded resources (degory/ghul#2477), and a source string works
under WebAssembly and single-file publish, where a file beside the assembly
does not.

## Tests

```sh
dotnet build
dotnet test unit-tests
dotnet ghul-test --use-dotnet-build integration-tests
```

The integration tests use the library as `dotnet build` last left it, rather
than each building it, since they run in parallel and would race on its
outputs.

The unit tests cover the font and the geometry. The integration test draws a
plot and compares it against `plot.png.expected`, which is the assertion that
matters: it is the whole picture, and it notices a stroke a fifth of a pixel
too thin.
