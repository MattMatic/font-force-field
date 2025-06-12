# Technical Overview
Having spent many hours with `collidoscope` (Simon Cozens), we had a wishlist of ideas for a similar tool.

What seemed like a simple geometric challenge turned into weeks of research and a lot of headaches.

JavaScript added another twist to the challenge, but WASM provides some good solutions.

# Curve Flattening
It is not possible to offset/inflate/outline/expand/buffer even a simple curve. This will always end up with a massively complex curve (e.g. 10th order!).

See: https://en.wikipedia.org/wiki/Parallel_curve
> Except in the case of a line or circle, the parallel curves have a more complicated mathematical structure than the progenitor curve.[1] For example, even if the progenitor curve is smooth, its offsets may not be so; this property is illustrated in the top figure, using a sine curve as progenitor curve.

So, we needed to flatten the OpenType glyph outlines into a polygon approximation. That could then be correctly offset to provide the necessary force-field regions.

Since HarfBuzzJS provides a `hb_font_draw_glyph` we can hook into the generation of cubic and quadratic bezier curves to flatten within JavaScript.

The blog post and library here provided the JS code:
- Blog: https://minus-ze.ro/posts/flattening-bezier-curves-and-arcs/
- Code: https://minus-ze.ro/flattening-bezier-curves-and-arcs.js

I reduced it into `js/flatbezier.js`.

So, the OpenType glyphs are flattened as we render them through HarfBuzz. And the list of polygon points is cached in `HarfBuzzForceFieldClass.glyphs.paths`.

# HarfBuzzJS
Rather than hacking into `hbjs.js` every time I needed to expand on it behaviour, I added this to the exports:
```js
    hooks: {
      exports: exports,
      addFunction: addFunction,
      removeFunction: removeFunction,
      utf8Decoder: utf8Decoder,
      Module: Module,
    }
```
Now I could build on the HarfBuzz behaviour directly in this tool (and others).

# Glyph definition - mark or base?
I tried as many OpenType JavaScript libraries as I could find. But `opentype.js` ended up being the simplest to extend. Not all libraries have the ability to read the `GDEF` table.

The `opentype.js` project includes `Layout.js` library (I think which was intended to be internal only), lets the tool query the OpenType GDEF table to determine the class of each glyph.

I got entangled in JS modules/imports vs scripts, so just created a subset of `Layout.js` in `opentype.layout.gdef.mjs`. (Must learn more JavaScript!)


# Outline Expansion
The `HarfBuzzForceFieldClass` caches the glyph path in `glyphs.paths.pathGlyph`.

As each unique GID is requested, the `glyphToPolyline` method will either return the cached entry, or will invoke HarfBuzz to:
- render the glyph with `hb_font_draw_glyph`
- flatten all curves in the callbacks
- generate the near/far/nearBase paths by using Clipper2's `InflatePaths64` method.

Only if the glyph has the mark flag does it calculate the 'NearBase' path.

# Determining Cursive Attachments
Initially, it seemed sensible to use the OpenType `GPOS` tables - seaching for the class=3 tables that define cursive attachments. This is how `collidoscope` works - just finding if a glyph has any cursive attachment point. However, `collidoscope` does not check whether these are attachments that are invoked - only that there is one.

Although I forked `opentype.js` and added most of the `GPOS` scanning code (which no JS library has, alas), a realisation came when considering the HarfBuzz debug tracing.

When using `hb_buffer_set_message_func`, a debug string callback occurs for every step of the HarfBuzz complex shaping. Using the `hooks` added to HarfBuzzJS, the tool can now find the messages `cursive attached`, and `attached mark` to discover _exactly_ which glyphs have been glued together!

NOTE: There's a little bit of juggling in the order for RTL scripts, since HarfBuzzJS hasn't exposed (through WASM) the buffer direction query functions.

There are two classes `CursiveSet` and `MarksSet` that keep track of which glyph indices have been glued together - and this provides a very solid way to refine the collision detection between individual glyphs.

# Clipper2 WASM memory leaks
Documentation? It could be a _lot_ clearer on this score. This took a LOT of debugging and digging.

It seems necessary to release memory for a path when complete, using `path.clear()`

However, after clearing, it also needs to be freed with `path.delete()`.
This, surprisingly, appears to be necessary even while just iterating and reading.

So, tried to wrap all this behaviour up in a class: `CPaths64`.


# Other libraries and tools considered
These are some of the libraries I came across while researching.
Might be useful!

- Paper.js: http://paperjs.org/
- paper-clipper: https://github.com/northamerican/paper-clipper
- JSTS: https://locationtech.github.io/jts/javadoc/
- shapely.js: https://github.com/chelm/shapely.js/tree/master
- Bezier flattening: https://gist.github.com/rlindsay/c55be560ec41144f521f
- SVG curves tool: https://github.com/xfbs/svgcurves
- simplify-path: https://github.com/mattdesl/simplify-path
- Pomax's lib-font: https://github.com/Pomax/lib-font

# Collidoscope
- collidoscope: https://github.com/googlefonts/collidoscope

Switching from collidoscope?

Collidoscope does its own permutations of glyphs, and I'd extended it to work on a concrete word list.

Suggest: create all the necessary Unicode permutations with Python etc, and use this word list for this tool.
