# font-force-field: Font Force Field Tool
Web based glyph collision and distance tool for complex OpenType fonts.

This tool is **completely offline** (i.e. nothing is sent to the cloud).

Having used Simon Cozen's excellent `collidoscope` project, but also been
badly bitten by Python library version nightmares, this was created as an
off-the-shelf tool.

While developing this tool, various other features became possible - like GID based
overrides for distances.

# Live
https://mattmatic.github.io/font-force-field

# Usage

- Drag and drop a font file onto the Font table cell (it will turn yellow)
    - Note: font name and version is shown
- Adhoc test:
  - Enter a word to show the glyphs, collisions, and other data
  - Use the sliders to set the force field sizes
- JSON definitions
  - You can define a JSON file for your optimum settings for the font under test
  - The JSON file can also include glyph ID overrides for special use cases
- Word list processing:
  - Drag and drop a .TXT file onto the Word List area on the right
  - (Recommend you use a JSON file to define the settings)
  - Click "Rate!" to iterate through all words and produce a list
  - When done, the "Save List" will download a text file of words and the rating outputs.
  - You can slo click "Font Compare Word List" to transfer everything over to the word list processing tool!

# Rating output abbreviations
The rating output that is used in the list:
- `GG#` Collision Glyph-to-Glyph. i.e. the glyphs genuinely overlap
- `cb#` Collision Base-to-Base Near Field
- `cx#` Collision Mark-to-Base Near Field
  - Either Near Field to Near Field
  - Or Near Field to Near-Base Field (depending on the relationship between the mark glyph and the base glyph)
- `cm#` Collision Mark-to-Mark Near Field
- `ca:####` Collision Area value. The `####` value is in design-units-squared
- `ok` Far Force Field might be ok
  - Where there is a gap in the Far Fields for the base, but the Far Fields of the marks join it all together
- `fb` Far force field shows a gap for the base glyphs
- `fm` Far force field shows a gap for the mark glyphs

HINT: if the output text _starts with_ `ok` then there are no collisions, and the word _might_ be ok if there is no `fm` in the output.

# JSON file definition

Suggestions:
- It is recommended you specify the font name and version.
- The `base` amd `mark` values **must** be defined as absolute (font design units)
- The GID overrides can use either percentages of the `base` or `mark` values, or can override in font design units
- Abbreviations:
  - `n` = Near Force Field
  - `nb` = Near Force Field for Mark-to-Base (or mark to cursive group)
  - `f` = Far Force Field

```json
{
  font:{
    name:'MyFontName',
    version:'Exact Version'
  },  // Warning if different

  // 'n'  = Near
  // 'f'  = Far
  // 'nb' = Near Base (mark to glyph or cursive group)

  base:{n:45, f:95},

  mark:{n:10, nb:5, f:100},

  // GID specific (use quotes around numbers)
  '643':{n:'120%',f:'200%'},  // Using percentages
  '644':{n:15,    f:50},      // Explicit values

  '454':{n:'80%', nb:'50%', f:40},  // Mixed. `nb` = mark-to-base near field

  // Named groups of GIDs for use with `bar` and `overlaps`
  groups: [
    consonants: [102,103,105],
    marks: [203,405,205],
  ],

  // For Indic scripts with joining top bar
  bar : [
    {
      // The area(s) can be either horizontal bars, or rectangles.
      // Several areas can be defined.
      // Only GIDs in this set remove the 'area' from collision checks
      area: [
        [600, 650],               // exclusion between y=600 and y=650
        [50, 300, 80, 400],       // Rectangle (50, 300) - (80, 400)
      ],
      gids: [56, 58, 70, 100],  // GIDs that are in the 'bar' set
      far: false,                 // false=bar is not in the far, true=bar included in far
    },
  ],

  // Array of arrays for GIDs that are allowed to overlap
  // Useful for Indic scripts with non-cursive glyphs that overlap in areas other than the 'bar' area
  overlaps: [                   
    [56, 59, 61],                 // i.e. 56 cannot overlap 57
    [57, 63, 75, 85],
    [consonants, 75, 85],         // The `group` called `consonant` and 75 and 85
  ],
}
```

# Technical Details
Please see: https://github.com/MattMatic/font-force-field/blob/main/TECHNICAL.md
